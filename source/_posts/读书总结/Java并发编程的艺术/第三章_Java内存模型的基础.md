---
title: 三、Java内存模型的基础
date: 2017-09-27 21:20:00
updated: 2017-09-27 21:20:00
comments: true
tags:
  - 并发
categories: 
  - [读书总结, Java并发编程的艺术]
permalink: concurrent-art/3.html    
---

# 1 Java 内存模型的基础

## 1.1 并发编程模型的两个关键问题

线程之间如何通信
线程之间如何同步（这里的线程是指并发执行的活动实体）。
通信是指线程之间以何种机制来交换信息。在命令式编程中，线程之间的通信机制有两种：

共享内存。线程之间共享程序的公共状态，通过写读内存中的公共状态进行隐式通信。
消息传递。线程之间没有公共状态，线程之间必须通过发送消息来显式进行通信。
同步是指程序中用于控制不同线程间操作发生相对顺序的机制。
在共享内存并发模型里，同步是显式进行的。必须显式指定某个方法或某段代码需要在线程之间互斥执行。
在消息传递的并发模型里，由于消息的发送必须在消息的接收之前，因此同步是隐式进行的。

Java 的并发采用的是共享内存模型，线程之间的通信对程序员完全透明。如果编写多线程不理解隐式进行的线程之间通信的工作机制，可能会遇到奇怪的内存可见性问题。

## 1.2 Java 内存模型的抽象结构

在 Java 中，所有实例域、静态域、数组元素都存储在堆内存中，堆内存在线程之间共享。局部变量（ Local Variables），方法定义参数和异常处理参数不会在线程之间共享， 它们不会有内存可见性问题，也不受内存模型的影响。

Java 线程之间的通信由 Java 内存模型（简称为 JMM ）控制， JMM 决定一个线程对共享变量的写入何时对另一个线程可见。从抽象的角度来看， JMM 定义了线程和主内存之间的抽象关系：线程之间的共享变量存储在主内存中，每个线程都有一个私有的本地内存，本地内存中存储了该线程以读/写共享变量的副本。 本地内存是 JMM 的一个抽象概念，并不真实存在。
![][1]

从上图看，如果线程A 与线程B 之间要通信的话，必须要经过下面 2 个步骤。

线程A 把本地内存A 更新过的共享变量刷新到主内存中去。
线程B 到主内存中去读取线程A 之前已更新过的共享变量。
如下图：
![][2]

本地内存A 和本地内存B 由主内存中共享变量 x 副本。假设初始时，这 3 个内存中的 x 值都为 0 。线程 A 在执行时，把更新后的 x 值（假设值为 1 ） 临时存放在自己的本地内存 A 中。当线程A 和线程B 需要通信时，线程A 首先会把自己本地内存中修改后的 x 值刷新到主内存中的 x 值变为了 1 。随后， 线程B 到主内存中去读取线程A 更新后的 x 值，此时线程B 的本地内存的 x 值也变为了 1 。

从整体来看，这两个步骤实质上是线程A 在向线程B 发送消息，而且这个通信过程必须要经过主内存。 JMM 通过控制主内存与每个线程的本地内存之间的交互， 来为程序员提供内存可见性保证。

## 1.3 从源代码到指令序列的重排序

执行程序，为了提高性能，编译器和处理器常常会对指令做重排序。重排序分三种：

编译器优化的重排序。
指令级并行的重排序。
内存系统的重排序。
JMM 属于语言级的内存模型，它确保在不同的编译器和不同的处理器平台之上，通过禁止特定类型的编译器重排序和处理器重排序，为程序员提供一致的内存可见性保证。

## 1.4 并发编程模型的分类

现在的处理器使用写缓存区临时保存向内存写入的数据。同时，通过以批处理的方式刷新写缓存区，以及合并写缓冲区对统一内存的多次写，减少对内存总线的占用。

## 1.5 happens-before 简介

在 JMM 中，如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须要存在 happens-before 关系。这里提到的两个操作既可以是在一个线程之内， 也可以是在不同线程之间。

与程序员密切相关的 happens-before 规则如下：
程序顺序规则： 一个线程中的每个操作， happens-before 于该线程中的任意后续操作。
监视器锁规则： 对一个锁的解锁， happens-before 与随后对这个锁的加锁。
volatile 变量规则： 对一个 volatile 域的写， happens-before 与任意后续对这个 volatile 域的读。
传递性： 如果 A happens-before B ,且 B happens-before C , 那么 A happens-before C 。
![][3]

一个 happens-before 规则对应一个或多个编译器和处理器重排序规则。它避免程序员为了理解 JMM 提供的内存可见性保证而去学习复杂的重排序规则。

# 2 重排序

重排序是指编译器和处理器为了优化程序性能而对指令序列进行重新排序的一种手段。

```java
class ReorderExample {
    int a = 0;
    bolean flag = false;
    public void writer() {
        a = 1;            // A
        flag = true;      // B
    }
    public void reader() {
        if (flag) {       // C
            int i = a * a; // D
        }
    }
}
```

假设有两个线程分别执行这两个方法，由于 a 变量和 flag 变量没有 happens-before 关系，因此有以下可能：
B->C->D->A。且由于 C 和 D 存在控制依赖关系，因此执行 reader() 方法的县横可以提前读取并计算 a * a ，然后把计算结果临时保存到一个名为重排序缓冲（Reorder Buffer，ROB）的硬件缓存中。 到为真时，就把该计算结果写入变量 i 中。因此，重排序在这里破坏了多线程程序的语义！

在单线程程序中，重排序不会改变执行结果。但是在多线程程序中，对存在控制依赖的操作重排序，可能会改变程序的执行结果。

# 3 顺序一致性

顺序一致性内存模型是一个理论参考模型，在设计的时候，处理器的内存模型和编程语言的内存模型都会以顺序一致性内存模型作为参考。

## 3.1 数据竞争与顺序一致性

JMM 保证，如果程序时正确同步的，程序的执行将具有顺序一致性——即程序的执行结果与该程序的顺序一致性内存模型中的执行结果相同。 这里的同步是指广义上的同步，包括对常用同步原语（synchronized 、 volatile 、 final）的正确使用。

## 3.2 顺序一致性内存模型

顺序一致性内存模型是一个被理想化了的理论参考模型：

一个线程中的所有操作必须按照程序的顺序来执行。
不管程序是否同步，所有线程都只能看到一个单一的曾作执行顺序。在顺序一致性内存模型中，每个操作都必须原子执行且立刻对所有线程可见。如下：
![][4]

这个内存通过一个左右摆动的开关可以连接任意一个线程，同时每一个线程必须按照程序的顺序来执行内存读/写操作。当多个线程并发执行时，图中的开关装置能把所有下城的所有内存读/写操作串行化。

## 3.3 同步程序的顺序一致性效果

```java
class SynchronizedExample {
    int a = 0;
    boolean synchronized void writer() {  // 获取锁
        a = 1;
        flag = true;
    }                                     // 释放锁
    public synchronized void reader() {   // 获取锁
        if (flag) {
            int i = a;
            ......
        }
    }                                     // 释放锁
}
```

下面是该程序在两个内存模型中执行时序对比图：
![][5]

可以得知， JMM 在具体实现上的基本方针为：在不改变（正确同步的）程序执行结果的前提下，尽可能地位编译器和处理器的优化打开方便之门。

## 3.4 总线的工作机制

![][6]

总线保证了处理器之间的串行化

# 4 volatile 的内存语义

实例代码如下：
```java
class VolatileFeaturesExample {
    volatile long i = 0L;     
    
    public void set(long i) {   
        this.i = i; 
    }
    
    public void getAndIncrement() {
        i++;
    }
    
    public long get() {
        return i;
    }
}
```

假设有多个线程分别调用上面程序的 3 个方法，这个程序在语义上和下面程序等价：

```java
class VolatileFeaturesExample {
    volatile long i = 0L;     
    
    public synchronized void set(long i) {   
        this.i = i; 
    }
    
    public void getAndIncrement() { 
        long temp = get();              // 调用已同步的读方法
        temp += 1L;
        set(temp);                      // 调用已同步的写方法
    }
    
    public long get() {
        return i;
    }
}
```

接着看第二个实例代码：

```java
public class TestVolatile {
    public volatile boolean flag = false;
    int a = 0;
    public void writer() {
        a = 1;
        flag = true;
    }
    public void reader() {
        if (flag) {
            System.out.println(a);
        }
    }
}
```

当写一个 volatile 变量时， JMM 会把该线程对应的本地内存中的共享变量值刷新到主内存。在读一个 volatile 变量时， JMM 会把该线程对应的本地内存置为无效。如下图：

![][7]

在读 flag 变量后，本地内存B 包含的值已经被置为无效。此时，线程B 必须从主内存中读取共享变量。线程B 的读取操作将导致本地内存B 与主内存中的共享变量的值一致。 

![][8]

volatile 内存语义的总结：

线程A 写一个 volatile 变量，实质上是线程A 向接下来将要读这个 volatile 变量的某个线程发出了（其对共享变量所做修改的）消息。
线程B 读一个 volatile 变量，实质上是线程B 接收了之前某个线程发出的（在读这个 volatile 变量之前对共享变量所做修改的）消息。
线程A 写一个 volatile 变量，随后线程B 读这个 volatile 变量，这个过程实质上是线程A 通过主内存向线程B 发送消息。

## 4.1 volatile 内存语义的实现

下面看看 JMM 如何实现 volatile 写/读的内存语义：

当第一个操作是 volatile 读时，不管第二个操作是什么，都不能重排序。
当第一个操作是 volatile 写时，第二个操作是 volatile 读时，不能重排序。
当第二个操作是 volatile 写时，不管第一个操作是什么，都不能重排序。

## 4.2 volatile 实践必读：https://www.ibm.com/developerworks/cn/java/j-jtp06197.html

[1]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/concurrent_art/3-1.png
[2]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/concurrent_art/3-2.png
[3]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/concurrent_art/3-3.png
[4]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/concurrent_art/3-4.png
[5]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/concurrent_art/3-5.png
[6]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/concurrent_art/3-6.png
[7]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/concurrent_art/3-7.png
[8]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/concurrent_art/3-8.png