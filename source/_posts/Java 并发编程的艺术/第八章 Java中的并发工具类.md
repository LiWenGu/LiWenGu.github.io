---
title: 八、Java中的并发工具类
date: 2017-11-07 20:07:00
updated: 2017-11-07 20:07:00
comments: true
categories: 
- 读书笔记
- Java并发编程的艺术  
permalink: concurrent-art/8.html    
---

# 1 等待多线程完成的 CountDownLatch

例如：解析一个 Excel 里多个 sheet 的数据，如果使用多线程，每个线程解析一个 sheet 里的数据，等到所有的 sheet 都解析完之后，程序提示解析完成。  
  
即，需要主线程等待所有线程完成 sheet 的解析操作，最简单的做法是使用 join() 方法，代码8-1：
```java
public class JoinCountDownLatchTest {
    public static void main(String[] args) throws Exception {
        Thread parser1 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("parser1 finish");
            }
        });
        Thread parser2 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("parser2 finish");
            }
        });
        parser1.start();
        parser2.start();
        parser1.join();
        parser2.join();
        System.out.println("all parser finish");
    }
}
```
join 用于让当前执行线程等待 join 线程执行结束。其实现原理是不听检查 join 线程是否存活，如果 join 线程存活则让当前线程永远等待。  
  
CountDownLacth 内部维护一个 int 类型的参数作为计数器，每次执行 countDown() 都会让计数器减1，await()只有当计数器为0的时候，才不会阻塞当前线程：
```java
public class CountDownLatchTest {

    static CountDownLatch c = new CountDownLatch(2);

    public static void main(String[] args) throws InterruptedException {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(1);
                c.countDown();
            }
        }).start();
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(2);
                c.countDown();
            }
        }).start();
        c.await();
        System.out.println("3");
    }
```

# 2 同步屏障 CyclicBarrier

## 2.1 CyclicBarrier 简介

字面意思：可循环使用（cyclic）的屏障（Barrier）。它要做的事情是，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，
屏障才会开门，所有被屏障拦截的线程才会继续运行。代码如下：
```java
public class CyclicBarrierTest2 {
    static CyclicBarrier c = new CyclicBarrier(2, new A());

    public static void main(String[] args) {
        Thread s = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    c.await();
                    Thread.sleep(1000); // 设置睡眠，输出：321，如果不设置，可能会是312
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
                System.out.println(1);
            }
        });

        s.start();

        try {
            c.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
        System.out.println(2);
    }

    static class A implements Runnable {

        @Override
        public void run() {
            System.out.println(3);
        }
    }
}
```

## 2.2 CyclicBarrier 的应用场景

用于多线程计算数据，最后合并计算结果的场景。例如，用一个 Excel 保存了用户所有银行流水，每个 Sheet 保存一个账户近一年的每笔银行流水，
现在需要统计用户的日均银行流水，先多线程处理每个 Sheet 的银行流水，都执行完之后，再用 barrierAction 计算线程结果。代码8-5如下：
```java
package com.lwg.current_art;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.*;

public class BankWaterService implements Runnable{

    /**
     * 创建 4 个屏障，当运行了 4 个await()后，才会运行第二参数。
     */
    private CyclicBarrier c = new CyclicBarrier(4, this);

    /**
     * 4 个sheet，创建 4 个线程的线程池
     */
    private Executor executor = Executors.newFixedThreadPool(4);

    /**
     * 保存每个线程的结果
     */
    private Map<String, Integer> sheetBankWaterCount = new ConcurrentHashMap<>();

    private void count() {
        for (int i = 0; i < 4; i++) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    sheetBankWaterCount.put(Thread.currentThread().getName(), 1);
                    System.out.println("size:" + sheetBankWaterCount.size());
                    try {
                        c.await();
                    } catch (InterruptedException | BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
    }

    @Override
    public void run() {
        int result = 0;
        for (Map.Entry<String, Integer> sheet: sheetBankWaterCount.entrySet()){
            result += sheet.getValue();
        }
        sheetBankWaterCount.put("result", result);
        System.out.println(result);
    }

    public static void main(String[] agrs) {
        BankWaterService bankWaterService = new BankWaterService();
        bankWaterService.count();
    }
}
```

##  2.3 CyclicBarrier 和 CountDownLatch 的区别

CountDownLatch 的计数器只能使用一次，而 CyclicBarrier 的计数器可以使用 reset() 方法重置。所以 CyclicBarrier 能处理更为复杂的业务。
一些 API 用法如下代码8-6：
```java
public class CyclicBarrierTest3 {

    static CyclicBarrier c = new CyclicBarrier(2);

    public static void main(String[] args) {
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    c.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }
        });
        thread.start();
        thread.interrupt();
        try {
            c.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            System.out.println(c.isBroken());
            e.printStackTrace();
        }
    }
}
```

## 2.3 控制并发线程数的 Semaphore

Semaphore(信号量)是用来控制同时访问特定资源的线程数量，通过协调各个线程，以保证合理的使用公共资源。  
  
把 Semaphore 比作是控制流量的红绿灯。比如xx马路要限制流量，只允许同时有一百辆车在这条路上行驶，其它的都必须在路口等待，所以前一百辆车会看到绿灯，
可以开进这条马路，后面的车会看到红灯，不能驶入xx马路，但是如果前一百辆中有5辆车已经离开了xx马路，那么后面就允许有5辆车驶入xx马路，
即车就是线程，驶入马路就是线程执行，离开马路就表示线程执行完成，看见红灯就表示线程被阻塞。

### 2.3.1 应用场景

Semaphore 可以用于做流量控制，特别是公用资源有限的应用场景，比如数据库连接。假设有一个需求，要读取几万个文件的数据，因为都是IO密集型人物，
我们可以启动几十个线程并发地读取，但是如果读到内存后，还需要存储到数据库中，而数据库的连接数只有10个，这时我们必须控制只有10个线程同时获取数据库连接保存数据，
否则会报错无法获取数据库连接。这个时候可以使用 Semaphore 做流量控制，如下代码8-7：
```java
public class SemaphoreTest {

    private static final int THREAD_COUNT = 30;

    private static ExecutorService threadPool = Executors.newFixedThreadPool(THREAD_COUNT);

    private static Semaphore s = new Semaphore(10);

    public static void main(String[] args) {
        for (int i = 0; i < THREAD_COUNT; i++) {
            threadPool.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        s.acquire();
                        System.out.println("save data");
                        s.release();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
        threadPool.shutdown();
    }
}
```

# 4 线程间交换数据的 Exchanger

Exchanger 提供一个同步点，在这个同步点，两个线程可以交换彼此的数据。如果第一个线程执行 exchange() 方法，它就会一直等待第二个线程也执行 exchange 方法，
当两个线程都到达同步点时，这时就可以交换数据。

## 4.1 应用场景

1. 遗传算法：选出两个人作为交配对象，交换两人的数据，并使用交叉规则得出2个交配结果。
2. 校对工作：对两个人工录入的文件进行校对。
```java
public class ExchangerTest {

    private static final Exchanger<String> exgr = new Exchanger<>();

    private static ExecutorService threadPool = Executors.newFixedThreadPool(2);

    public static void main(String[] args) {
        threadPool.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    String A = "银行流水A";
                    exgr.exchange(A);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        threadPool.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    String B = "银行流水B";
                    String A = exgr.exchange(B);
                    System.out.println("A和B数据是否一致：" + A.equals(B) + ".A录入的是：" + A + ".B录入的是：" + B);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        threadPool.shutdown();
    }
}
```

# 5 本章小结

1. CountDownLatch->CyclicBarrier：都是等待某些运行到某个点后，才执行后面的方法，但是 CyclicBarrier提供的 API 更多适合更复杂的场景。
2. Semaphore：控制并发数，即创建了30个线程，但是并发最多可以设置为10。
3. Exchanger：线程间交换数据，在同步点处，A线程可以获得B线程的数据。



