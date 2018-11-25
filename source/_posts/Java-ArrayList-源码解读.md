---
title: Java8 ArrayList 源码解读
date: 2018-11-12 21:07:33
categories:
- Java
tags:
- Java
- ArrayList
---


*推荐阅读时间：10分钟*

### 简介
ArrayList 是日常开发中很常见的集合类型，在 Java 集合中相对容易阅读。它是基于数组实现的一种列表，读取、修改的时间复杂度很小（O(1)）, 插入、remove 时时间复杂度为O(n)。ArrayList 可以存放 null 值，列表清空就是通过把所有的元素置为 null 实现的。

### Arrays.copyOf() 和 System.arraycopy()

首先我们先看下代码里反复出现的两个方法：Arrays.copyOf() 和 System.arraycopy()。其实 ` public static <T> T[] copyOf(T[] original, int newLength)` 是通过调用后者实现的，输入待拷贝的数组和要返回数组的长度，拷贝出一个新的数组。而`public static native void arraycopy(Object src,  int  srcPos, Object dest, int destPos,int length)`是真正执行拷贝的方法。它是个 native 方法，五个参数分别代表 待拷贝数组、待拷贝数组的起始位置、目标数组、目标数组的插入位置、拷贝的长度。每次拷贝都是全量拷贝，因此容量变化的操作较多时，会对它造成性能影响。


### RandomAccess

ArrayList 实现了 RandomAccess 接口表示支持快速随机访问，将使用 for 循环查找元素。如果没有实现该接口（如 LinkedList），在查找时，只能通过 迭代器 进行查找，查找速度要低于前者。

Java doc 中具体解释如下：
```
 * <pre>
 *     for (int i=0, n=list.size(); i &lt; n; i++)
 *         list.get(i);
 * </pre>
 * runs faster than this loop:
 * <pre>
 *     for (Iterator i=list.iterator(); i.hasNext(); )
 *         i.next();
 * </pre>
```


### ⭐注释版源码⭐


```

public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    private static final long serialVersionUID = 8683452581122892189L;

    /**
     * 默认的初始化容量：10
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * Object[]，用来存储列表元素
     */
    transient Object[] elementData; // non-private to simplify nested class access
    /**
     * 按默认容量创建的空列表共享的存储对象
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    /**
     * 指定容量创建的空列表共享的存储对象
     */
    private static final Object[]
            EMPTY_ELEMENTDATA = {};
    /**
     * 列表中元素数量
     */
    private int size;

    /**
     * 构造函数，指定容量大小
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            //如果指定容量为0，使用 EMPTY_ELEMENTDATA
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                    initialCapacity);
        }
    }

    /**
     * 无参构造函数
     */
    public ArrayList() {
        //使用 DEFAULTCAPACITY_EMPTY_ELEMENTDATA
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    /**
     * 构造函数，入参为一个集合类型
     */
    public ArrayList(Collection<? extends E> c) {
        //调用 Collection 的 toArray() 方法，转换为 Object[]
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray 方法可能会转换出错，导致生成 Object[] 失败
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }

    /**
     * 按实际元素数量重新申请存储空间以减少内存使用
     */
    public void trimToSize() {
        //修改次数。记录该 ArrayList 对象修改次数，防止并发执行修改操作导致数据不一致。
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
                    ? EMPTY_ELEMENTDATA
                    : Arrays.copyOf(elementData, size);
        }
    }

    /**
     * 确保容量足够，如果容量不够就扩容
     */
    public void ensureCapacity(int minCapacity) {
        //如果是按默认容量创建的空 ArrayList,则当指定的容量超过10时，才会扩容
        int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
                // any size if not default element table
                ? 0
                // larger than default for default empty table. It's already supposed to be at default size.
                : DEFAULT_CAPACITY;

        if (minCapacity > minExpand) {
            ensureExplicitCapacity(minCapacity);
        }
    }

    //计算最小容量
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }

    //计算出需要扩容的最小容量然后确保增加到该容量
    private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }

    //确保增加到指定的容量
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

    /**
     * 元素最大数量。因为有的虚拟机预留了用于保存数组对象大小等信息的元数据，故减去了8位。
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    /**
     * 实际执行 扩容方法
     */
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        //扩容到原来的 1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        //如果扩容后仍小于要最小容量则直接取最小容量作为新的容量
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        //如果扩容后大于最大允许的容量，则执行 hugeCapacity
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    /**
     * 求扩容容量，如果实际指定的最小容量超过 MAX_ARRAY_SIZE ，
     * 则取 Integer.MAX_VALUE。否则取 MAX_ARRAY_SIZE。
     */
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
                Integer.MAX_VALUE :
                MAX_ARRAY_SIZE;
    }

    /**
     * 返回实际元素个数
     */
    public int size() {
        return size;
    }

    /**
     * 是否为空
     */
    public boolean isEmpty() {
        return size == 0;
    }

    /**
     * 是否包含一个元素
     */
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }

    /**
     * 求指定元素首次出现的下标。-1 表示不存在
     */
    public int indexOf(Object o) {
        //单独考虑所查元素为 null 时的情况
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }

    /**
     * 指定元素最后一次出现的下标。靠从后面遍历来实现的。
     */
    public int lastIndexOf(Object o) {
        if (o == null) {
            for (int i = size-1; i >= 0; i--)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = size-1; i >= 0; i--)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }

    /**
     * 浅拷贝，只拷贝一个新的数组，元素未拷贝。
     */
    public Object clone() {
        try {
            ArrayList<?> v = (ArrayList<?>) super.clone();
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError(e);
        }
    }

    /**
     * 转换为 Object[]
     */
    public Object[] toArray() {
        return Arrays.copyOf(elementData, size);
    }

    /**
     * 转换为指定类型数组。
     * 注意：若指定数组类型不是列表元素的超类，则会报 ArrayStoreException 异常。
     * 若指定数组为 null ,会报空指针异常。
     */

    @SuppressWarnings("unchecked")
    public <T> T[] toArray(T[] a) {
        if (a.length < size)
            // Make a new array of a's runtime type, but my contents:
            return (T[]) Arrays.copyOf(elementData, size, a.getClass());
        System.arraycopy(elementData, 0, a, 0, size);
        if (a.length > size)
            a[size] = null;
        return a;
    }

    // Positional Access Operations

    @SuppressWarnings("unchecked")
    E elementData(int index) {
        return (E) elementData[index];
    }

    /**
     * get
     */
    public E get(int index) {
        //检测下标是否越界
        rangeCheck(index);

        return elementData(index);
    }

    /**
     * set
     */
    public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }

    /**
     * add
     */
    public boolean add(E e) {
        //先判断容量是否足够
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                size - index);
        elementData[index] = element;
        size++;
    }

    /**
     * 根据下标删除
     */
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);
        //需要移动的元素数量
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                    numMoved);
        //将数组最后一个元素设为 null,通过GC机制自动去回收空间
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }

    /**
     * 根据元素删除
     */
    public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

    /**
     * 与 remove(int index) 方法基本一致，只是没有下标越界检查、不返回旧值。
     * 提升删除性能，作为私有方法。
     */
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                    numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }

    /**
     * 所有元素全部置为 null
     */
    public void clear() {
        modCount++;

        // clear to let GC do its work
        for (int i = 0; i < size; i++)
            elementData[i] = null;

        size = 0;
    }

    /**
     * 追加一个集合的元素
     */
    public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }

    /**
     * 插入一个集合的元素
     */
    public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount

        int numMoved = size - index;
        if (numMoved > 0)
            System.arraycopy(elementData, index, elementData, index + numNew,
                    numMoved);

        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }

    /**
     * 删除指定下标范围的元素
     */
    protected void removeRange(int fromIndex, int toIndex) {
        modCount++;
        int numMoved = size - toIndex;
        System.arraycopy(elementData, toIndex, elementData, fromIndex,
                numMoved);

        // clear to let GC do its work
        int newSize = size - (toIndex-fromIndex);
        for (int i = newSize; i < size; i++) {
            elementData[i] = null;
        }
        size = newSize;
    }

    /**
     * 检测下标是否越界
     */
    private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    /**
     * add 和 addAll 方法中 检测下标是否越界。
     */
    private void rangeCheckForAdd(int index) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    private String outOfBoundsMsg(int index) {
        return "Index: "+index+", Size: "+size;
    }

    /**
     * 删除在集合中出现的元素
     * 可能会报 ClassCastException 和 空指针异常
     */
    public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, false);
    }

    /**
     * 保留在集合中存在的元素
     * 可能会报 ClassCastException 和 空指针异常
     */
    public boolean retainAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, true);
    }

    private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;
        boolean modified = false;
        try {
            //遍历列表，根据 complement，选择是否保留元素
            for (; r < size; r++)
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            //如果遍历过程中报错了，将剩余未遍历的元素追加到不完整的新列表的后面
            if (r != size) {
                System.arraycopy(elementData, r,
                        elementData, w,
                        size - r);
                w += size - r;
            }
            //如果列表有被修改，则将无效存储位置置为 null
            if (w != size) {
                // clear to let GC do its work
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
        //返回是否做了修改
        return modified;
    }

    /**
     * 对象序列化函数
     */
    private void writeObject(java.io.ObjectOutputStream s)
            throws java.io.IOException{
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        //执行默认的序列化函数，将除 elementData[] 外的属性序列化
        s.defaultWriteObject();

        // Write out size as capacity for behavioural compatibility with clone()
        // 写入 size
        s.writeInt(size);

        // Write out all elements in the proper order.
        // 将 elementData[] 中的元素序列化进去
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }
        //序列化过程中对象被修改，则报 ConcurrentModificationException 异常
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

    /**
     * 反序列化函数
     */
    private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;

        // Read in size, and any hidden stuff
        // 反序列化出除 elementData[] 外的属性
        s.defaultReadObject();

        // Read in capacity
        // 读出 size ,可忽略
        s.readInt(); // ignored

        if (size > 0) {
            // be like clone(), allocate array based upon size not capacity
            // 计算出实际容量
            int capacity = calculateCapacity(elementData, size);
            // 检查是否转换为数组类型，容量是否小于0。此处实际上不需要第一个参数。
            SharedSecrets.getJavaOISAccess().checkArray(s, Object[].class, capacity);
            ensureCapacityInternal(size);


            Object[] a = elementData;
            // Read in all elements in the proper order.
            // 执行反序列化
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }
    }

    /**
     * 返回一个迭代器 ListIterator
     */
    public ListIterator<E> listIterator() {
        return new ListItr(0);
    }

    /**
     * 返回一个迭代器 ListIterator，指定初始迭代位置
     */
    public ListIterator<E> listIterator(int index) {
        if (index < 0 || index > size)
            throw new IndexOutOfBoundsException("Index: "+index);
        return new ListItr(index);
    }

    /**
     * 返回一个迭代器 Iterator
     */
    public Iterator<E> iterator() {
        return new Itr();
    }


    /**
     * 获取子list
     */
    public List<E> subList(int fromIndex, int toIndex) {
        subListRangeCheck(fromIndex, toIndex, size);
        return new SubList(this, 0, fromIndex, toIndex);
    }

    //子list范围检查
    static void subListRangeCheck(int fromIndex, int toIndex, int size) {
        if (fromIndex < 0)
            throw new IndexOutOfBoundsException("fromIndex = " + fromIndex);
        if (toIndex > size)
            throw new IndexOutOfBoundsException("toIndex = " + toIndex);
        if (fromIndex > toIndex)
            throw new IllegalArgumentException("fromIndex(" + fromIndex +
                    ") > toIndex(" + toIndex + ")");
    }


    // Java8 新加，供函数式编程遍历
    @Override
    public void forEach(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        final int expectedModCount = modCount;
        @SuppressWarnings("unchecked")
        final E[] elementData = (E[]) this.elementData;
        final int size = this.size;
        for (int i=0; modCount == expectedModCount && i < size; i++) {
            action.accept(elementData[i]);
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

    /**
     * 返回一个 Spliterator 对象。
     * Spliterator 是 Java8 新加的、可分割的迭代器(splitable iterator)，对于并行处理的能力大大增强。
     */
    @Override
    public Spliterator<E> spliterator() {
        return new ArrayListSpliterator<>(this, 0, -1, 0);
    }

    /**
     * Predicate 是Java8 新加的类，可以理解为一个用来断言的类。
     * 该方法应该是为了函数式编程新加的。
     */
    @Override
    public boolean removeIf(Predicate<? super E> filter) {
        Objects.requireNonNull(filter);
        // figure out which elements are to be removed
        // any exception thrown from the filter predicate at this stage
        // will leave the collection unmodified
        int removeCount = 0;
        final BitSet removeSet = new BitSet(size);
        final int expectedModCount = modCount;
        final int size = this.size;
        for (int i=0; modCount == expectedModCount && i < size; i++) {
            @SuppressWarnings("unchecked")
            final E element = (E) elementData[i];
            if (filter.test(element)) {
                removeSet.set(i);
                removeCount++;
            }
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }

        // shift surviving elements left over the spaces left by removed elements
        final boolean anyToRemove = removeCount > 0;
        if (anyToRemove) {
            final int newSize = size - removeCount;
            for (int i=0, j=0; (i < size) && (j < newSize); i++, j++) {
                i = removeSet.nextClearBit(i);
                elementData[j] = elementData[i];
            }
            for (int k=newSize; k < size; k++) {
                elementData[k] = null;  // Let gc do its work
            }
            this.size = newSize;
            if (modCount != expectedModCount) {
                throw new ConcurrentModificationException();
            }
            modCount++;
        }

        return anyToRemove;
    }

    /**
     * Java8 新加，可以按指定规则替换所有元素。
     * UnaryOperator 实现了 Function 接口，可以接收一个入参，处理后返回。
     */
    @Override
    @SuppressWarnings("unchecked")
    public void replaceAll(UnaryOperator<E> operator) {
        Objects.requireNonNull(operator);
        final int expectedModCount = modCount;
        final int size = this.size;
        for (int i=0; modCount == expectedModCount && i < size; i++) {
            elementData[i] = operator.apply((E) elementData[i]);
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
        modCount++;
    }

    /**
     * 排序，Comparator 指定排序规则
     */
    @Override
    @SuppressWarnings("unchecked")
    public void sort(Comparator<? super E> c) {
        final int expectedModCount = modCount;
        //调用 Arrays 的排序方法，瞄了一眼，很复杂的样子...
        Arrays.sort((E[]) elementData, 0, size, c);
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
        modCount++;
    }
}
```

### writeObject 和 readObject
ArrayList 实现了 Serializable 接口，所以对象会被序列化。而存放元素的 elementData 中可能会存在元素数量比数组容量小很多的情况，序列化时就会造成大量的空间浪费，因此通过实现 writeObject 和 readObject 方法，即可重新定义序列化与反序列化的规则。ArrayList 在 elementData 前加上了 transient 取消其默认序列化规则，其他属性则执行默认的规则。

### 迭代器
`iterator()`方法会返回一个 Iterator 迭代器，遍历时较常见。
源码如下：

```
/**
 * AbstractList.Itr 的优化版 迭代器
 */
private class Itr implements Iterator<E> {
    // 当前下标
    int cursor;       // index of next element to return
    // 上一个元素的下标，-1 表示还没有上一个元素
    int lastRet = -1; // index of last element returned; -1 if no such
    int expectedModCount = modCount;

    Itr() {}

    public boolean hasNext() {
        return cursor != size;
    }

    @SuppressWarnings("unchecked")
    public E next() {
        checkForComodification();
        int i = cursor;
        if (i >= size)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i + 1;
        return (E) elementData[lastRet = i];
    }

    public void remove() {
        //
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();

        try {
            ArrayList.this.remove(lastRet);
            cursor = lastRet;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }
    //Java8 新加的 遍历方法，供函数式编程使用
    @Override
    @SuppressWarnings("unchecked")
    public void forEachRemaining(Consumer<? super E> consumer) {
        Objects.requireNonNull(consumer);
        final int size = ArrayList.this.size;
        int i = cursor;
        if (i >= size) {
            return;
        }
        final Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length) {
            throw new ConcurrentModificationException();
        }
        while (i != size && modCount == expectedModCount) {
            consumer.accept((E) elementData[i++]);
        }
        // update once at end of iteration to reduce heap write traffic
        cursor = i;
        lastRet = i - 1;
        checkForComodification();
    }

    // 检查是否被修改过
    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}

```

`listIterator(int index)`、`listIterator()`这两个方法返回的是 ListIterator 迭代器，与 Iterator 相比，它支持反向遍历和 add() 方法，比较容易理解，不再赘述。

此外，还有一个 `spliterator()`方法，它返回的是一个 Java8 新加的 Spliterator 迭代器。Spliterator 是一个可分割迭代器(splitable iterator)，为了并行遍历元素而设计。如果有机会我们再分析它。


### sublist

ArrayList 提供的`public List<E> subList(int fromIndex, int toIndex)`方法允许返回一个子list。
根据注释得知：
1. 该方法返回的是父list的一个视图，从fromIndex（包含），到toIndex（不包含）。fromIndex=toIndex 表示子list为空
2. 父子list做的非结构性修改（non-structural changes）都会影响到彼此：所谓的“非结构性修改”，是指不涉及到list的大小改变的修改。相反，结构性修改，指改变了list大小的修改。
3. 对于结构性修改，子list的所有操作都会反映到父list上。但父list的修改将会导致返回的子list失效。
4. tips：删除list中的某段数据的方法：`list.subList(from, to).clear();`




*********************

觉得有点收获的同学可以在手机上点击这个[链接](https://m.luckincoffee.com/invited/register?activityNo=NR201801030001&inviteCode=8lB4421fo_6iv_eidD4_Fg%3D%3D&secondfrom=0&title=%E4%BB%8A%E5%A4%A9%E6%98%9F%E6%9C%9F%E4%B8%89%EF%BC%8C%E8%AF%B7%E4%BD%A0%E5%96%9D%E6%9D%AF%E5%85%8D%E8%B4%B9%E5%A4%A7%E5%B8%88%E5%92%96%E5%95%A1%EF%BC%8C%E6%96%B9%E6%A1%88%E4%B8%80%E7%A8%BF%E8%BF%87&timestamp=1542129222502&from=singlemessage) 免费领取一杯咖啡（瑞幸咖啡券，使用后我也得一张😃）



