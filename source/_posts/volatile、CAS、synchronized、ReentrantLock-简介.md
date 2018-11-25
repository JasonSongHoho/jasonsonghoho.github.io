---
title: volatile、CAS、synchronized、ReentrantLock 简介
date: 2018-11-24 12:23:02
categories:
- Java
tags:
- 多线程
- volatile
- CAS
- synchronized
- ReentrantLock
---


*推荐阅读时间：10分钟*

### 简介
volatile、CAS、synchronized、ReentrantLock 都是多线程中需要理解的重要知识，本文把它们放一起对比下，做个简单的介绍，为后面分析concurrent包源码打好基础。
其中 volatile 和 CAS 是用来保证对变量的操作的线程安全性，synchronized 和 Lock 是用来保证多个操作的线程安全性。

### 一个实验
我们先通过一个小实验来简单了解下他们的使用方法和区别。

```
public class AtomicLab {
    private static final int LOOP_TIME = 500;
    private static final Object LOCK = new Object();
    private static Integer availableProcessors = Runtime.getRuntime().availableProcessors();
    private static Lock lock = new ReentrantLock();

    private static Integer i0 = 0;
    private static volatile Integer i1 = 0;
    private static Integer i2 = 0;
    private static AtomicInteger i3 = new AtomicInteger();
    private static Integer i4 = 0;

    public static void main(String[] args) throws InterruptedException {

        ThreadFactory threadFactory = new ThreadFactoryBuilder().setNameFormat("jasonLab-%d").build();
        ExecutorService service = new ThreadPoolExecutor(availableProcessors + 1, availableProcessors * 2,
                60L, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>(5000), threadFactory);
        for (int i = 0; i < 100; i++) {
            service.execute(new TestThread0());
            service.execute(new TestThread1());
            service.execute(new TestThread2());
            service.execute(new TestThread3());
            service.execute(new TestThread4());
        }
        Thread.sleep(1000L);
        System.out.println("i0 result is " + i0 + " , equal 50000 : " + (i0 == 50000));
        System.out.println("i1 result is " + i1 + " , equal 50000 : " + (i1 == 50000));
        System.out.println("i2 result is " + i2 + " , equal 50000 : " + (i2 == 50000));
        System.out.println("i3 result is " + i3 + " , equal 50000 : " + (i3.get() == 50000));
        System.out.println("i4 result is " + i4 + " , equal 50000 : " + (i4 == 50000));

        service.shutdown();
    }

    static class TestThread0 implements Runnable {

        @Override
        public void run() {
            for (int i = 0; i < LOOP_TIME; i++) {
                i0 += 1;
            }
        }
    }

    static class TestThread1 implements Runnable {

        @Override
        public void run() {
            for (int i = 0; i < LOOP_TIME; i++) {
                i1++;
            }
        }
    }

    static class TestThread2 implements Runnable {

        @Override
        public void run() {
            for (int i = 0; i < LOOP_TIME; i++) {
                synchronized (LOCK) {
                    i2 += 1;
                }
            }
        }
    }

    static class TestThread3 implements Runnable {

        @Override
        public void run() {
            for (int i = 0; i < LOOP_TIME; i++) {
                i3.getAndAdd(1);
            }
        }
    }

    static class TestThread4 implements Runnable {

        @Override
        public void run() {
            lock.lock();
            try {
                for (int i = 0; i < LOOP_TIME; i++) {
                    i4++;
                }
            } finally {
                lock.unlock();
            }

        }
    }
}
```

运行上述 main 方法，一个可能的结果如下：
```
i0 result is 48646 , equal 50000 : false
i1 result is 48509 , equal 50000 : false
i2 result is 50000 , equal 50000 : true
i3 result is 50000 , equal 50000 : true
i4 result is 50000 , equal 50000 : true

```

上述实验是计算 100 个线程同时对同一个 i 进行`i++`操作的累加结果。
我们知道，`i++`操作其实分：读（getI()）、改（i=i+1）、写（setI(i)）三步进行的。
对于 i0，这三个操作都不具备原子性保证，所以多线程下难免会发生数据丢失的问题。而至于i1-i4，其实分别用到了标题中的四个知识点，我们依次介绍下它们。

### volatile
i1 被 volatile 修饰，它是 Java 中的关键字，它修饰的变量具有可见性和原子性的特点。
#### 可见性和原子性
可见性：如果一个变量具有可见性，可以理解为任意时刻得到的都是该变量的最新值。
原子性：指对该变量的操作是不可中断的。即使是在多个线程一起执行的时候，一个操作一旦开始，就不会被其它线程干扰。

#### volatile 实现原理
volatile 修饰的变量在进行操作时，会在汇编代码中加上 `Lock` 前缀，这将导致两件事情：
1. 所有处理器不会在本地内存中记录该变量，而是直接写到共享内存中。
2. 所有处理器在读取该变量时，都直接从共享内存中读取。

#### 结果分析
根据实现原理，我们可以得知：**对 volatile 变量的读或写都可以保证原子性**。也就是上面的第一步和第三步是原子性的操作，但是第二步修改操作时却不能保证。
当一个线程执行修改操作时，其他线程可能已经执行过写入操作了，所以当该线程执行写入操作时，就覆盖了前面的写入操作，导致数据丢失。

### CAS
我们先看下 i3，可以看到它使用了原子更新整型：`AtomicInteger`，我们在进行累加时，使用了它的`getAndAdd()`方法。
这个方法其实最终调用了`Unsafe.compareAndSwapInt()`方法，这是个 native 方法，依赖 CAS（CompareAndSwap）原理实现。

#### CAS 实现原理
CAS 的实现使用了处理器提供的 `CMPXCHG`指令，这个指令也带有`Lock`前缀，在进行 CAS 操作时，会锁住相应的内存区域，其他不能操作相应内存区域的线程在外面循环进行尝试，实现多线程原子性。
进行 CAS 时，需要对三个值进行操作：现在的值、预期的值、要替换的值。只有当预期的值和当前值一致时，才会进行修改。

#### ABA问题
CAS 操作可能会出现这样的问题：变量的值原来是A，被其他线程修改为了B，后来又被修改回A，当该线程进行CAS操作时，发现预期值与当前值一致，进行了修改。而其实变量已经被修改过了，这样就可能会导致其他的问题。JDK1.5开始，提供了`AtomicStampedReference`类来解决这个问题，变量会加一个类似乐观锁的版本号：1A-2B-3A。这样就可以准确的判断变量是否被修改过了。

#### 结果分析
根据原理，我们可以得知针对 i3 的每次修改都是原子性的，没啥好说的～
synchronized 和 ReentrantLock 也不再进行结果分析。

#### 延伸
volatile 和 CAS 在 Java 中举足轻重。借一张图表示 Java concurrent 包的实现。

{% asset_img concurrent包.png concurrent 包实现  %}

### synchronized

synchronized 是 Java 提供的一个关键字，用来锁住一个对象，被锁的对象任意时刻只能被一个线程访问（同一个线程可以加多个锁进行重复访问）。
synchronized 修饰不同的地方，加的锁的类型也不一样：
1. 修饰非静态方法，锁的是该方法所在的实例对象。
2. 修饰静态方法，锁的是该类的类对象。
3. 修饰代码块时，锁的是所指定的对象。


#### 实现原理
任何一个对象都有一个 monitor 与之关联，当 monitor 被持有后，它就将处于锁定状态。synchronized 就是通过获取和释放 monitor 实现的。

{% asset_img synchronized.png synchronized（重量级锁）原理%}

#### 锁状态
大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低，Java6 开始，引入了`偏向锁`和`轻量级`锁的概念。
##### 偏向锁
获取到锁后，锁默认处于`偏向锁`状态，在锁对象的对象头中储存一个线程ID，当下次该线程尝试获取该锁时，不需要进行循环CAS取锁，只需要检测偏向锁的线程ID是否与之一致即可。
当多个线程对同一个锁竞争激烈时，偏向锁会升级为`轻量级锁`。
##### 轻量级锁
加锁：
线程在执行同步块之前，JVM会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的 Mark Word 复制到锁记录中，官方称为 Displaced Mark Word。
然后线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。
解锁：
轻量级解锁时，会使用原子的CAS操作将Displaced Mark Word替换回到对象头，如果成功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。

轻量级锁能提高程序同步性能的依据是“对于绝大部分的锁，在整个同步周期内都是不存在竞争的”，这是一个经验数据。如果没有竞争，轻量级锁使用CAS操作避免了使用互斥量的开销，
但如果存在锁竞争，除了互斥量的开销外，还额外发生了CAS操作，因此在有竞争的情况下，轻量级锁会比传统的重量级锁更慢。

### ReentrantLock
ReentrantLock 实现了 Lock 接口，也是 JDK 中该接口的唯一实现。Lock 接口是在Java5新增的，提供了与 synchronized 相似的功能。

#### 与 synchronized 的区别
1. `ReentrantLock`可以显示的进行加锁和解锁。
2. `ReentrantLock`可中断的获取锁。
3. `ReentrantLock`可以提供公平锁。
4. `ReentrantLock`可以提供超时等待机制。

#### 实现原理
ReentrantLock 的实现依赖于 AbstractQueueSynchronizer(AQS),它是实现锁或其他同步组件的基础框架。
AQS 内部维护了一个同步状态变量和一个同步队列，获取到该同步状态的线程视为获取到锁；获取失败的线程连同它的等待状态信息会被构造成加入到同步队列中，并阻塞它。
当同步状态被释放时，同步队列中的首节点会被唤醒尝试去获取同步状态。

#### 读写锁
如果一段代码中大部分时间都在执行读操作，多个读操作同时进行不会影响线程安全性，这时前面提到的独占锁明显会影响多线程的读取性能。
`ReentrantReadWriteLock`是一个读写锁，多个获取了读锁之间的线程可以同步执行；而写锁不可以和读/写锁同步执行。
读写锁锁降级：一个线程在获取了写锁后，有获取了读锁，在释放写锁后，就变成了只获取了读锁，即锁降级。

#### Condition
ReentrantLock 使用 Condition 的 `await()`、`signal()`、`signalAll()`方法分别代替 Object 的`wait()`、`notify()`、`notifyAll()` 方法。


### 结语
以上是关于这四者的简单介绍，为了后面的系列内容做下铺垫，想要了解详情可以参考更多书籍、资料。


*********************
{% asset_img img-6806f.gif %}

