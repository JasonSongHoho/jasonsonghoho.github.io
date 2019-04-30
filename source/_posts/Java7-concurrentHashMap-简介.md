---
title: Java7 ConcurrentHashMap 简介
date: 2019-04-30 18:52:14
categories:
- Java
tags:
- Java
- ConcurrentHashMap
---



*推荐阅读时间：15分钟*


### 简介
Java7 ConcurrentHashMap 是线程安全的 HashMap。与 HashTable 的区别是，支持多个线程并发访问，吞吐量高。
结构如下：
{% asset_img concurrentHashMap.png %}

如图，ConcurrentHashMap 就是 Segment 数组，每个 Segment 可以理解为一个 HashMap。同一时间，每个 Segment 只允许一个线程访问。Segment 的数量在初始化后不再允许更改，但是每个 Segment 的长度是可以改变的。



### Unsafe 类
Unsafe 类是 jdk 提供的一个类，这个类提供了一些绕开 JVM 的更底层功能，基于它的实现可以提高效率。但它是一把双刃剑：正如它的名字所讲，它是 Unsafe 的，它所分配的内存需要手动 free（不被 GC 回收）。
ConcurrentHashMap 中反复用到了其中的几个方法，如下：

```
  /***
   * Sets the value of the object field at the specified offset in the
   * supplied object to the given value.  This is an ordered or lazy
   * version of <code>putObjectVolatile(Object,long,Object)</code>, which
   * doesn't guarantee the immediate visibility of the change to other
   * threads.  It is only really useful where the object field is
   * <code>volatile</code>, and is thus expected to change unexpectedly.
   * 设置obj对象中offset偏移地址对应的object型field的值为指定值。这是一个有序或者
   * 有延迟的<code>putObjectVolatile</cdoe>方法，并且不保证值的改变被其他线程立
   * 即看到。只有在field被<code>volatile</code>修饰并且期望被意外修改的时候
   * 使用才有用。
   *
   * @param obj the object containing the field to modify.
   *    包含需要修改field的对象
   * @param offset the offset of the object field within <code>obj</code>.
   *       <code>obj</code>中long型field的偏移量
   * @param value the new value of the field.
   *      field将被设置的新值
   */
  public native void putOrderedObject(Object obj, long offset, Object value);
```

```
 /***
   * Retrieves the value of the object field at the specified offset in the
   * supplied object with volatile load semantics.
   * 获取obj对象中offset偏移地址对应的object型field的值,支持volatile load语义。
   *
   * @param obj the object containing the field to read.
   *    包含需要去读取的field的对象
   * @param offset the offset of the object field within <code>obj</code>.
   *       <code>obj</code>中object型field的偏移量
   */
  public native Object getObjectVolatile(Object obj, long offset);
```


```
  /***
   * Compares the value of the object field at the specified offset
   * in the supplied object with the given expected value, and updates
   * it if they match.  The operation of this method should be atomic,
   * thus providing an uninterruptible way of updating an object field.
   * 在obj的offset位置比较object field和期望的值，如果相同则更新。这个方法
   * 的操作应该是原子的，因此提供了一种不可中断的方式更新object field。
   *
   * @param obj the object containing the field to modify.
   *    包含要修改field的对象
   * @param offset the offset of the object field within <code>obj</code>.
   *         <code>obj</code>中object型field的偏移量
   * @param expect the expected value of the field.
   *               希望field中存在的值
   * @param update the new value of the field if it equals <code>expect</code>.
   *               如果期望值expect与field的当前值相同，设置filed的值为这个新值
   * @return true if the field was changed.
   *              如果field的值被更改
   */
  public native boolean compareAndSwapObject(Object obj, long offset, Object expect, Object update);
```


### 初始化
见注释：
```
public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    int sshift = 0;
    //ssize 是不小于预设并发度且为2的幂数的最小值
    int ssize = 1;
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    //segmentShift 和 segmentMask 是用于计算 Segment 位置的,int j =(hash >>> segmentShift) & segmentMask 。
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    // c 是每个 Segment 的容量
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    while (cap < c)
        cap <<= 1;
    // 创建 Segment 数组，并初始化 Segment[0]。
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
```


### get
很简单：
```
public V get(Object key) {
    Segment<K,V> s;
    HashEntry<K,V>[] tab;
    int h = hash(key);
    //u 是该 Segment 在数组中的位置
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    //Segment 不为空且 Segment 的 HashEntry 数组不为空时，遍历链表找到值返回；否则返回 null 。
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

### put
put 较复杂：
```
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key);
    int j = (hash >>> segmentShift) & segmentMask;
    //如果要插入的 Segment 为 null，执行 ensureSegment ,初始化它。
    if ((s = (Segment<K,V>)UNSAFE.getObject(segments, (j << SSHIFT) + SBASE)) == null)
        s = ensureSegment(j);
    //调用 Segment 的 put 方法
    return s.put(key, hash, value, false);
}
```

ensureSegment 如下：

```
private Segment<K,V> ensureSegment(int k) {
    final Segment<K,V>[] ss = this.segments;
    long u = (k << SSHIFT) + SBASE; // raw offset
    Segment<K,V> seg;
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
        //使用 Segment[0] 的容量和负载因子初始化该 Segment
        Segment<K,V> proto = ss[0];
        int cap = proto.table.length;
        float lf = proto.loadFactor;
        int threshold = (int)(cap * lf);
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        //使用 CAS 将该 Segment 初始化
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) { // recheck
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                    break;
            }
        }
    }
    return seg;
}
```

实际执行插入的 Segment 的 put 方法：
```
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    //尝试获取一次锁，获取到往下走；否则执行 scanAndLockForPut 方法来获取锁
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> first = entryAt(tab, index);
        //获取到头节点，如果链表为空，插入该值直接返回 null；
        //否则进行遍历，找到相应 key 则进行修改,返回旧值，没找到则在最后插入该值然后返回 null。
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}

```

用于获取锁的 scanAndLockForPut：
```
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1; // negative while locating node
    //循环调用 tryLock() 尝试获取锁（自旋），当对某一元素的自旋次数超过一定次数时将被阻塞
    while (!tryLock()) {
        HashEntry<K,V> f; // to recheck first below
        //自旋开始时，先遍历找到对应的元素
        if (retries < 0) {
            if (e == null) {
                if (node == null) // speculatively create node
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0;
            }
            else if (key.equals(e.key))
                retries = 0;
            else
                e = e.next;
        }
        //MAX_SCAN_RETRIES:最大加锁尝试次数，单核下为1；多核下为64。
        //如果已经超过最大尝试次数则停止自旋，当前线程被阻塞，休眠一直到该锁可以获取。
        else if (++retries > MAX_SCAN_RETRIES) {
            lock();
            break;
        }
        //如果 retries 为偶数且该表头元素已经更改则重新开始自旋
        else if ((retries & 1) == 0 && (f = entryForHash(this, hash)) != first) {
            e = first = f; // re-traverse if entry changed
            retries = -1;
        }
    }
    return node;
}

```

### 扩容
实际执行扩容的是 Segment 的 rehash 方法：
```
private void rehash(HashEntry<K,V> node) {
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    //容量翻倍
    int newCapacity = oldCapacity << 1;
    threshold = (int)(newCapacity * loadFactor);
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    int sizeMask = newCapacity - 1;
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            int idx = e.hash & sizeMask;
            if (next == null)   //  Single node on list
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                //第一次遍历，找出最后面有哪些连续的元素扩容后会在相同的节点上
                //根据概率来讲，平均后面 5/6 的数据都会在相同的节点上
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                newTable[lastIdx] = lastRun;
                // 克隆分界节点（lastrun）之前的所有元素
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    int nodeIndex = node.hash & sizeMask; // add the new node
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}

```


### isEmpty()、size()
isEmpty 与 size 两个方法与 HashMap 有些区别。因为它要遍历的对象是所有 Segment。直接对所有 Segment 加锁遍历十分影响性能，因此这两个方法用了共同的特殊技巧来实现。

isEmpty 方法会不加锁遍历两次，只要两次获取到的 modCount(修改次数)不一致或某个 Segment 的元素不为0，则认为不为空。
```
public boolean isEmpty() {
    long sum = 0L;
    final Segment<K,V>[] segments = this.segments;
    for (int j = 0; j < segments.length; ++j) {
        Segment<K,V> seg = segmentAt(segments, j);
        if (seg != null) {
            if (seg.count != 0)
                return false;
            sum += seg.modCount;
        }
    }
    if (sum != 0L) { // recheck unless no modifications
        for (int j = 0; j < segments.length; ++j) {
            Segment<K,V> seg = segmentAt(segments, j);
            if (seg != null) {
                if (seg.count != 0)
                    return false;
                sum -= seg.modCount;
            }
        }
        if (sum != 0L)
            return false;
    }
    return true;
}
```

size 方法也会先执行两次不加锁的遍历，若两次获取的 size 的大小一致则直接返回，否则加锁重新获取 size。
```
public int size() {
    // Try a few times to get accurate count. On failure due to
    // continuous async changes in table, resort to locking.
    final Segment<K,V>[] segments = this.segments;
    int size;
    boolean overflow; // true if size overflows 32 bits
    long sum;         // sum of modCounts
    long last = 0L;   // previous sum
    int retries = -1; // first iteration isn't retry
    try {
        for (;;) {
            //前两次遍历获取的 size 的大小不一致，则加锁获取 size。
            if (retries++ == RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    ensureSegment(j).lock(); // force creation
            }
            sum = 0L;
            size = 0;
            overflow = false;
            for (int j = 0; j < segments.length; ++j) {
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount;
                    int c = seg.count;
                    if (c < 0 || (size += c) < 0)
                        overflow = true;
                }
            }
            if (sum == last)
                break;
            last = sum;
        }
    } finally {
        //如果加锁了，需要进行解锁
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();
        }
    }
    //如果 size 已超过32位，则取 Integer.MAX_VALUE，否则取实际的 size。
    return overflow ? Integer.MAX_VALUE : size;
}

```


个人陋见难免疏漏，不足之处还请多多指教。😄

江湖路远，我们下期再会。
