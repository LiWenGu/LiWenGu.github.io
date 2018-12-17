---
title: ThreadLocal 的心得
date: 2017-12-22 01:24:00
updated: 2017-12-22 01:24:00
tags:
  - threadlocal
categories: 
  - 小结
  - threadlocal
comments: true
permalink: tech/threadlocal_1.html 
---

在《架构探险——从零开始架构》中，第四章的自己实现 ThreadLocal 感悟：  
ThreadLocal 中虽然使用了 Map 进行保存线程变量，但是为了防止引入锁（Map 的多线程访问）影响性能，从而使用让不同的 Thread 保存不同的 Map（ThreadLoaclMap）实例，这样不同的Thread 有不同的 ThreadLocalMap 实例，就不用考虑锁的问题。   
另外为了避免内存泄漏、回收不及时等问题，从而让 ThreadLocalMap 的 key 使用弱引用。  
同时，为了保证当 key 为 null 时，value 无法正常释放时，在每次 set 时，都会遍历 key ，当 key 为 null 则会执行 replaceStaleEntry()，即将 key 为 null 的 value 值也置为 null，从而来让其回收。    
这里讲解更加详细：http://www.jasongj.com/java/threadlocal/  
这是原理：![][1]

[1]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/others/threadlocal.png
