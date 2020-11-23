---
title: 五、Java中的锁
date: 2017-11-07 20:07:00
updated: 2017-11-07 20:07:00
comments: true
tags:
  - java并发
categories: 
  - 理论
  - Java并发编程的艺术
permalink: concurrent-art/5.html    
---

# 1 Lock 接口

# 2. 队列同步器

队列同步器 AbstractQueueSynchronizer （简称同步器），是用来构建锁或者其它同步组件的基础框架，它使用了一个 int 成员变量表示同步状态，
通过内置的 FIFO 队列完成资源获取线程的排队工作。  
  
## 2.1 队列同步器的接口与示例

同步器是实现锁（也可以是任意同步组件）的关键，在锁的实现中聚合同步器，利用同步器实现锁的语义。可以这样理解二者的关系：锁是面向使用者，
它定义了使用者与锁交互的接口，隐藏了实现细节；同步器面向的是锁的实现者，它简化了锁的实现方式，屏蔽了同步状态管理、线程的排队、等待与唤醒等底层操作。
锁和同步器很好地隔离了使用者和实现者所需关注的领域。  
  
只有掌握了同步器的工作原理才能更加深入地理解并发包中其它的并发组件，所以下面通过一个独占锁的实例来深入了解一下同步器的工作原理。  
  
独占锁就是在同一时刻只能由一个线程获取到锁，而其它获取锁的线程只能处于同步队列中等待，只有获取锁的线程释放了锁，后继的线程才能获取锁，代码5-2如下：  
```java
public class Mutex implements Lock{
    // 静态内部类，自定义同步器
    private static class Sync extends AbstractQueuedSynchronizer {

        // 是否处于占用状态
        @Override
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        @Override
        public boolean tryAcquire(int acquires) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int releases) {
            if (getState() == 0) throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        Condition newCondition() {
            return new ConditionObject();
        }
    }

    private final Sync sync = new Sync();

    @Override
    public void lock() {
        sync.acquire(1);
    }

    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(time));
    }

    @Override
    public void unlock() {
        sync.release(1);
    }

    public boolean isLocked() {
        return sync.isHeldExclusively();
    }

    public boolean hasQueuedThreads(){
        return sync.hasQueuedThreads();
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }
}
```

## 2.2 队列同步器的实现分析

分析同步器如何完成线程同步的，包括：同步队列、独占式同步状态获取与释放、共享式同步状态获取与释放以及超时获取同步状态等同步器的核心数据结构与模块方法。

### 2.2.1 同步队列

当前线程获取同步状态失败时，同步器会将当前线程以及等待状态等信息构造成一个节点 （Node） 并将其加入同步队列，同时会阻塞当前线程，当同步状态释放时，
会把首节点中的线程唤醒，使其再次尝试获取同步状态。  
![][1]  
试想，当一个线程成功地获取了同步状态（锁），其它线程将无法获取到同步状态，转而被构造称为节点并加入到同步队列中，而这个加入队列的过程必须要
保证线程安全，因此同步器提供了一个基于 CAS 线程的设置尾节点的方法： `compareAndSetTail(Node expect, Node update)` ，它需要传递当前“认为”
的尾节点和当前节点，只有设置成功后，当前节点才正式与之前的尾节点建立关联。  
![][2]  
首节点是获取同步状态成功的节点，首节点的线程在释放同步状态时，将会唤醒后继节点，而后继节点将会在获取同步状态成功时将自己设置为首节点：
![][3]  
设置首节点是通过获取同步状态成功的线程来完成的，由于只有一个线程能够成功获取到同步状态，因此设置头节点的方法并不需要使用 CAS 来保证。

### 2.2.2 独占式同步状态获取与释放

通过调用同步器的 acquire(int arg) 方法可以获取同步状态，该方法对中断不敏感，也就是由于线程获取同步状态失败后进入同步队列中，后续对线程
进行中断操作时，线程不会从同步队列中移出，源码如下：
```java
public final void acquire(int arg) {
    if(!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

# 3 重入锁

ReenTrantLock，支持重入的锁。

## 3.1 实现重进入

这个特性需要解决两个问题：
1. 线程再次获取锁。锁需要去识别获取锁额线程是否为当前占据锁的线程，如果是，则再次成功获取。
2. 锁的最终释放。线程重复 n 次获取了锁，随后在第 n 次释放锁后，其它线程能够获取到锁。要求对获取进行计数自增，释放时计数自减。  
  
获取锁源码：
```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
释放锁源码：
```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

## 3.2 公平与非公平获取锁的区别

如果一个锁是公平的，那么锁的获取顺序就应该符合请求的绝对时间顺序，也就是FIFO。  
源码如下：
```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
其中相对于非公平锁的获取，多了一个 `hasQueuedPredecessors()` 判断，判断是否当前节点是否有前驱节点的判断。  
  
如果有 n 个线程争夺锁。对于公平锁来说，保证了每个线程都可以获取锁，即有 n 次线程切换。而对于非公平锁来说，有可能有的线程能获取多次锁，
有的线程根本获取不到锁，线程切换次数是小于 n 的。因此减去上下文的切换时间，非公平锁的效率更高，所以 `ReentrantLock` 默认为非公平锁。 

# 4 读写锁

在没有读写锁支持的时候（Java5之前），使用 Java 的等待通知机制，就是当写操作开始时，所有晚于写操作的读操作均会进入等待状态，只有写操作完成并行通知后，
所有等待的读操作才能继续执行（写操作之间依靠 synchronized 关键字进行同步）。  
  
一般来说，读写锁的性能都会比排他锁好（ReentrantLock 也是一种排他锁），因为大多数场景读是多于写的。Java并发包提供读写锁的实现是 `ReentrantReadWriteLock`。

## 4.1 读写锁的接口与示例

通过缓存示例说明读写锁的使用方式，代码5-16：
```java
public class Cache {

    static Map<String, Object> map = new HashMap<>();

    static ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

    static Lock r = rwl.readLock();

    static Lock w = rwl.writeLock();

    public static final Object get(String key) {
        r.lock();
        try {
            return map.get(key);
        } finally {
            r.unlock();
        }
    }
    
    public static final Object put(String key, String value) {
        w.lock();
        try {
            return map.put(key, value);
        } finally {
            w.unlock();
        }
    }
    
    public static final void clear() {
        w.lock();
        try {
            map.clear();
        } finally {
            w.unlock();
        }
    }
}
```
使用非线程安全的 `HashMap` 作为缓存的实现。使用读写锁提升读操作的并发性，并简化了编程方式。

## 4.2 读写锁的实现分析

### 4.2.1 读写状态的设计

读写锁依赖自定义同步器来实现同步功能，而读写状态就是其同步器的同步状态。同步状态表示锁被一个线程重复获取的次数，而读写锁的自定义同步器需要在同步状态
（一个整型变量）上维护多个读线程和一个写线程的状态，使得该状态的设计成为读写锁实现的关键。  
  
如果在一个整型变量上维护多种状态，就一定需要“按位切割使用”这个变量，读写锁将变量切分成了两个部分，高16位表示读，低16位表示写。  
![][4]  
当前图的同步状态表示一个线程已经获取了写锁，且重进入了两次，同时也连续获取了两次读锁。读写锁是如何迅速确定读和写各自的状态呢？答案是通过位运算。
假设当前同步状态值为 S ，写状态等于 S&0x0000FFFF（将高16位全部抹去），读状态等于 S>>>16（无符号补0右移16位）。当写状态增加1时，等于S+1,，
当读状态增加1时，等于S+(1<<16)，也就是S+0x00010000。  
  
根据状态的划分能得出一个推论：S不等于0时，当写状态（S&0x0000FFFF）等于0时，则读状态（S>>>16）大于0，即读锁已被获取。

### 4.2.2 写锁的获取与释放

写锁是一个支持重进入的排它锁。如果当前线程已经获取了写锁，则增加写状态。如果当前线程在获取写锁时，读锁已经被获取或者该线程不是已经获取写锁的线程，
则当前线程进入等待状态，获取锁的源码如下：
```java
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // 存在读锁或者当前获取线程不是已经获取写锁的线程
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

### 4.2.3 读锁的获取与释放

读锁是一个支持重进入的共享锁，能被多个线程同时获取，在没有其它写线程访问时，读锁总会被成功获取。如果当前线程已经获取了读锁，则增加读状态。
如果当前线程在获取读锁时，写锁已被其他线程获取，则进入等待状态。获取读锁的实现从 Java5 到 Java6 变得复杂许多，主要原因是新增了一些功能，例如
`getReadHoldCount()` 方法，作用是返回当前线程获取读锁的次数。读状态是所有线程获取读锁次数的综合，而每个线程各自获取读锁的次数只能选择
保存在 `ThreadLocal` 中，由线程自身维护，这使得获取读锁的实现变得复杂。因此，这里将获取读锁的代码做了删减，保留必要的部分：
```java
protected final int tryAcquireShared(int unused) {
    for (;;) {
        int c = getState();
        int nextc = c + (1 << 16);
        if (nextc < c)
            throw new Error("Maximum lock count exceeded");
        if (exclusiveCount(c) != 0 && owner != Thread.currentThread())
            return -1;
        if (compareAndSetState(c, nextc))
            return 1;
    }
}
```
如果其它线程已经获取了写锁，则当前线程获取读锁失败，进入等待状态。如果当前线程获取了写锁或者写锁未被获取，则当前线程增加读状态，成功获取读锁。

### 4.2.4 锁降级

锁降级指的是写锁降级称为读锁。获取写锁-释放写锁-获取读锁，这种分段完成的过程不能称之为锁降级。锁降级是指：获取写锁-获取读锁-释放写锁。



[1]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/concurrent_art/5_1.png
[2]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/concurrent_art/5_2.png
[3]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/concurrent_art/5_3.png
[4]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/concurrent_art/5_4.png


