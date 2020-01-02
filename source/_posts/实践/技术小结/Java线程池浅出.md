---
title: Java线程池浅出
date: 2020-01-02 18:32:00
updated: 2020-01-02 18:32:00
tags:
  - threadPool
categories: 
  - 实践
  - 技术小结
comments: true
permalink: tech/java_threadpool_learing.html    
---

# 0. 背景

线程池一直对其中的线程销毁机制有点模糊，在网上找了很多文章，终于找到：https://juejin.im/post/5c33400c6fb9a049fe35503b。  
该文通过源码入手分析，写的真不错！  
而本文两个目的：一是能让自己能给别人讲懂任务如何的执行以及讲懂线程如何的销毁。

# 1. 线程池状态

1. Running：接收新任务，处理队列任务
2. SHUTDOWN：不接受新任务，但会处理队列任务
3. STOP：不接受新任务，不处理队列任务，会中断所有处理中的任务
4. TIDYING：所有任务都被终结，有效线程为0.会触发terminated()方法
5. TERMINATED：当terminated()方法执行结束

# 2. 线程池变量

1. ctl：AtomicInteger 类型。高3位存储线程池状态，低29位存储当前线程数量。workerCountOf(c) 返回当前线程数量。runStateOf(c) 返回当前线程池状态。
2. worker：一个 Worker 代表一个线程。
3. workers：HashSet，管理 worker，即管理当前线程池的所有线程。
4. allowCoreThreadTimeOut：超时时间 keepAliveTime 是否对核心线程也有效，即核心线程在空闲时是否也要销毁？

# 3. 线程池方法

1. submit：将 Runnable 包装为 RunnableFuture，实际会调用 execute。
2. execute：
工作线程数小于核心线程数时，新建核心线程执行任务。  
大于核心线程数时，将任务添加进等待队列。  
队列满时，创建非核心线程执行任务。  
工作线程数大于最大线程数时，执行拒绝策略。  

# 4. 任务的执行

每个任务会包装成 Worker 执行，在执行时，先执行 addWorker() 方法，添加入 workers，接着执行 Worker.run。  
每个 worker 执行都通过死循环调用 getTask() 方法堵塞，这个方法又是通过死循环不断的从队列中取任务执行。

# 5. 线程的销毁

线程销毁通过 getTask() 方法以及 processWorkerExit() 这两个方法实现，前者通过 queue 的 poll 做定时堵塞，后者通过 workers.remove 来做线程的移除。