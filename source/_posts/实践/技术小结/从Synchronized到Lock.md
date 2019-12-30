---
title: 从Synchronized到Lock
date: 2019-12-30 23:54:00
updated: 2019-12-30 23:54:00
tags:
  - 锁
  - Synchronized
  - AQS
  - Lock
  - CAS
  - ConcurrentHashMap
categories: 
  - 实践
  - 技术小结
comments: true
permalink: tech/from_Synchronized_to_Lock.html    
---

# 0. 背景

一步一步的复习锁，从 Synchronized 到 AQS，其实还有 volatile，但是它是可见性和原子性的保证，和锁还是有本质区别的，就没有纳入了。

# 1. Synchronized

对象头：每个对象都有一个对象头，里面存储了锁类型（是否偏向锁、轻量锁、重量锁）、GC 标记、对象分代年龄等。
monitor：ObjectMonitor 对象，也叫监视器，每个对象都有一个对应的监视器，里面包括处于阻塞该对象锁的线程以及处于等待该对象锁的线程（前者是排队获取锁、后者是 wait 方法）。


每个对象都有对应的一个 monitor，哪个线程获取到 monitor，就得到了该对象的锁。线程实际通过操作对象头来控制对象的锁。是否为重量锁、轻量锁，是通过对象头的标识来确定锁类型，他们可以转换，即从轻量锁转换成重量级锁等。  
Synchronized 对应两条指令：monitorenter、monitorexit。这两条指令表明某个线程持有某个 monitor，以及释放某个 monitor。 
其中 wait/notify 表示：将对象当前被持有的锁放入等待队列，因此这个操作是需要提前获取锁，即需要在 Synchronized 代码块内操作。notify 表示唤醒对象被持有的锁。

# 2. CAS

CAS：compare and swap，基于 UnSafe 类来实现，UnSafe 通过指针操作与操作系统交互，最终依赖操作系统的 CAS 指令完成。
虽然是无锁技术，但是并不代表性能比有锁技术高，因为当锁竞争严重时，会不断的在循环中 CAS 操作，性能会很低。  
>long、double 占用8字节，也就是64比特，而在 32 位操作系统对 64 位的数据读写要分两步。这个没有做试验。

# 3. ConcurrentHashMap

从 1.7 的 Segment + HashEntry 到 1.8 的 Synchronized + CAS + 红黑树，比之前的分段锁更加细粒，头结点使用的 CAS 插入，冲突失败后使用 Synchronized 获取头结点锁进行后续操作。

# 4. AQS

AQS 核心在于使用的 CAS 来改动 status 状态值，作为是否持有锁，可重入锁等。以及额外包括了等待队列 CLH，该等待队列用于阻塞线程后的排队容器，因此可以设置排队是否公平。而 ReentrantLock 本质就是 AQS 的一个实现。