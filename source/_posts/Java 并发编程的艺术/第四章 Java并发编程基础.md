---
title: 四、Java并发编程基础
date: 2017-11-07 20:07:00
updated: 2017-11-07 20:07:00
comments: true
categories: 
- 读书笔记
- Java并发编程的艺术
permalink: concurrent-art/4.html    
---

# 1 线程简介

## 1.1 什么是线程

操作系统运行一个程序时，会为其创建一个进程。而操作系统调度的最小单元是线程，也叫轻量级进程（ Light Weight Process），
在一个进程里可以创建多个线程，这些线程都拥有各自的计数器、堆栈和局部变量等属性，并且能够访问共享的内存变量。  
处理器在这些线程上高速切换，让使用者感觉到这些线程在同时执行。
```java
public class MultiThread{
    public static void main(String[] args) {
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(false, false);
        for(ThreadInfo threadInfo : threadInfos) {
            System.out.println(threadInfo.getThreadId() + "," + threadInfo.getThreadName());
        }
        /**
         6,Monitor Ctrl-Break
         5,Attach Listener
         4,Signal Dispatcher
         3,Finalizer
         2,Reference Handler
         1,main
         */
    } 
}
```

## 1.2 为什么要使用多线程

### 1.2.1 更多的处理器核心

程序运行过程中能够创建多个线程，而一个线程在一个时刻只能运行在一个处理器核心上。试想一下，一个单线程程序在运行时只能使用一个处理器
核心，那么再多的处理器核心加入也无法显著提升该程序的执行效率。相反，如果该程序使用多线程技术，将计算逻辑分配到多个处理器核心上，
就会显著减少程序的处理时间，并且随着更多处理器核心的加入而变得更有效率。

### 1.2.2 更快的响应时间

编写一些较为复杂的代码，例如，一笔订单的创建，它包括插入订单数据、生成订单快照、发送邮件通知卖家和记录货品销售数量等。这么多业务
操作，如何能够让其更快地完成呢？

### 1.2.3 更好的编程模型

Java 为多线程编程提供了良好、考究并且一致的编程模型，使开发人员更加专注于问题的解决。

## 1.3 线程优先级

现代操作系统基于采用时分的形式调度运行的线程，操作系统会分出一个个时间片，线程会分配到若干时间片，当线程的时间片用完了就会发生线程
调度，并等待下次分配。线程分配到的时间片多少也就决定了线程使用处理器资源的多少，而线程优先级就是决定线程需要多或者少分配一些处理器
资源的线程属性。  
  
在 Java 线程中，通过一个整型成员变量 priority 来控制优先级，优先级的范围从 1~10 ，在线程构建的时候可以通过 setPriority(int) 方法
来修改优先级，默认优先级是 5 ，优先级高的线程分配时间片的数量要多余优先级低的线程。设置线程优先级时，针对频繁阻塞的线程需要设置较高优先级，
而偏重计算（需要较多 CPU 时间或者偏运算）的线程则设置较低的优先级。笔者在 JDK 1.8 的 WIN 10 环境：
```java
public class Priority {

    private static volatile boolean notStart = true;

    private static volatile boolean notEnd = true;

    public static void main(String[] args) throws InterruptedException {
        List<Job> jobs = new ArrayList();
        for (int i = 0; i < 10; i++) {
            int priority = i < 5 ? Thread.MIN_PRIORITY : Thread.MAX_PRIORITY;
            Job job = new Job(priority);
            jobs.add(job);
            Thread thread = new Thread(job, "Thread:" + i);
            thread.setPriority(priority);
            thread.start();
        }
        notStart = false;
        TimeUnit.SECONDS.sleep(2);
        notEnd = false;
        for (Job job : jobs) {
            System.out.println("Job Priority：" + job.priority + ", Count：" + job.jobCount);
        }
    }

    static class Job implements Runnable {

        private int priority;
        private long jobCount;

        public Job(int priority) {
            this.priority = priority;
        }

        @Override
        public void run() {
            while (notStart) {
                Thread.yield();
            }
            while (notEnd) {
                Thread.yield();
                jobCount++;
            }
        }
    }
}
```
输出：
```java
Job Priority：1, Count：16772
Job Priority：1, Count：16761
Job Priority：1, Count：16761
Job Priority：1, Count：16758
Job Priority：1, Count：16757
Job Priority：10, Count：756747
Job Priority：10, Count：757594
Job Priority：10, Count：757263
Job Priority：10, Count：759519
Job Priority：10, Count：760287
```

## 1.4 线程的状态

![][1]

## 1.5 Daemon 线程

Daemon 线程是一种支持型线程，因为它主要被用作程序中后台调度以及支持性工作。这意味着，当一个 Java 虚拟机中不存在非 Daemon 线程的时候，
Java 虚拟机将会退出。可以通过调用 Thread.setDaemon(true) 将线程设置为 Daemon 线程。
>Daemon 属性需要在启动线程之前设置，不能在启动线程之后设置。

Daemon 线程被用作完成支持性工作，但是在 Java 虚拟机退出时 Daemon 线程中的 finally 块并不一定会执行，如下：
```java
public class Daemon {
    public static void main(String[] args) {
        Thread thread = new Thread(new DaemonRunner(), "DaemonRunner");
        thread.setDaemon(true);
        thread.start();
    }

    static class DaemonRunner implements Runnable {

        @Override
        public void run() {
            try {
                SleepUtils.second(10);
            } finally {
                System.out.println("DaemonThread finally run.");
            }
        }
    }
}
```
最终没有任何的输出，mian 线程在启动了线程 DaemonRunner 之后随着 main 方法执行完毕而终止，而此时 Java 虚拟机中已经灭有非 Daemon 线程，
虚拟机需要退出。 Java 虚拟机中的所有 Daemon 线程都需要立即终止，因此 DaemonRunner 立即终止，但是 DaemonRunner 中的 finally 块并没有执行。

# 2 启动和终止线程

## 2.1 理解中断

中断可以理解为线程的一个标识位属性，它标识一个运行中的线程是否被其它线程进行了中断操作。线程通过方法 `isInterrupted` 来进行判断是否被中断，
也可以调用静态方法 `Thread.interrupted()` 对当前线程的中断标识位进行复位。下面的例子中，创建了两个线程， SleepThread 和 BusyThread ，
前者不停地睡眠，后者一直运行，然后对这两个线程分别进行中断操作，观察二者的中断标识位。

```java
public class Interrupted {
    public static void main(String[] args) throws InterruptedException {
        Thread sleepThread = new Thread(new SleepRunner(), "SleepThread");
        sleepThread.setDaemon(true);

        Thread busyThread = new Thread(new BusyRunner(), "BusyThread");
        busyThread.setDaemon(true);

        sleepThread.start();
        busyThread.start();

        Thread.sleep(5 * 1000);

        sleepThread.interrupt();
        busyThread.interrupt();

        System.out.println("SleepThread interrupted is " + sleepThread.isInterrupted());
        System.out.println("BusyThread interrupted is " + busyThread.isInterrupted());

        Thread.sleep(2 * 1000);


        System.out.println("SleepThread interrupted is " + sleepThread.isInterrupted());
        System.out.println("BusyThread interrupted is " + busyThread.isInterrupted());
        // SleepThread interrupted is false
        // BusyThread interrupted is true
        
        Thread.sleep(2 * 1000);
    }

    static class SleepRunner implements Runnable {

        @Override
        public void run() {
            while (true) {
                try {
                    Thread.sleep(10 * 1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    static class BusyRunner implements Runnable {

        @Override
        public void run() {
            while (true) {
            }
        }
    }
}
```
线程 SleepThread 其中断标识位被清除了，而一直忙碌运作的线程 BusyThread ，其中断标识位没有被清除。

## 2.2 安全地终止线程

```java
public class Shutdown {
    
    public static void main(String[] args) throws InterruptedException {
        Runner one = new Runner();
        Thread countThread = new Thread(one, "countThread");
        countThread.start();
        
        Thread.sleep(1 * 1000);
        
        countThread.interrupt();
        
        Runner two = new Runner();
        countThread = new Thread(two, "CountThread");
        countThread.start();
        
        Thread.sleep(1 * 1000);
        
        two.cancel();
    }
    
    private static class Runner implements Runnable {
        private long i ;
        private volatile boolean on = true;
        
        @Override
        public void run() {
            while(on && !Thread.currentThread().isInterrupted()) {
                i++;
            }
        }
        
        private void cancel() {
            on = false;
        }
    }
}
```
使用 `Thread.interrupted()` 或一个 boolean 来进行中断。

# 3. 线程间通信

## 3.1 volatile 和 synchronized 关键字

每个执行的线程拥有一份拷贝，这样做的目的是加速程序的执行，所以程序在执行过程中，一个线程看到的变量并不一定是最新的。  
  
关键字 `volatile` 就是告知程序任何对该变量的访问均需要从共享内存中获取，而对它的改变必须同步刷新回共享内存，保证所有线程对变量访问的可见性。  
  
举个例子：定义一个表示程序是否运行的成员变量 boolean on = true ，另一个线程可能执行了关闭动作 on = false ，这里涉及多个线程对变量的访问，
因此需要定义称为 volatile boolean on = true ，这样其他线程对它改变时，所以线程都会感知，因为所有对 on　变量的访问和修改都需要以共享内存
为准。  
  
关键字 synchronized 主要确保多个线程在同一时刻，只能由一个线程处于方法或者同步块中，保证了线程对变量访问的可见性和排他性。  
  
使用 javap 工具分析 synchronized 关键字的实现细节，实例4-10：
```java
public class Synchronized {
    public static void main(String[] args) {
        synchronized (Synchronized.class) {

        }
        m();
    }

    public static synchronized void m() {

    }
}
```
`javap -v Synchronized.class` :同步块的前面和后面分别有 monitorenter 和 monitorexit 指令，而同步方法依靠方法修饰符 ACC_SYNCHRONIZED 来完成。  
其本质是对一个对象的监视器(monitor)进行获取，这个获取过程是排他的，即同一时刻只有一个线程获取到由 sychronized 所保护对象的监视器。  
  
任意一个对象都拥有自己的监视器，执行方法的线程必须先获取到该对象的监视器，没有获取到的线程将会阻塞在入口处，进入 BLOCKED 状态。  
![][2]

## 3.2 等待/通知机制

一个线程修改了一个对象的值，另一个线程感知到了变化，进行相应的操作，整个过程开始于一个线程，最终执行又是另一个线程。前者是生产者，后者就是消费者。
在功能上进行了解耦，“做什么”和“怎么做”。在 Java 实现类似的功能：  
  
简单的方法就是让消费者线程不断地循环检查变量是否符合预期，如下的消费者：
```java
while (value != desire) {
    Thread.sleep(1000); // 防止过快的“无效”尝试
}
doSomething();
```
存在的问题：
1. 难以确保及时性。在睡眠时，基本不消耗处理器资源，但是如果睡得过久，就不能及时发现条件已经变化。
2. 难以降低开销。如果降低睡眠时间，比如休眠 1 毫秒，这样消费者能更加迅速地发生条件变化，但是却可能消耗更多的处理器资源，造成了无端的浪费。

可以通过等待/通知的相关方法是任意 Java 对象都具备的，因为这些方法被定义在 Object ，方法和描述如下表所示：

| 方法名称         | 描述                                                                                                        |
| :---------------| :----------------------------------------------------------------------------------------------------------|
| notyfy()        | 通知一个在对象上等待的线程，使其从 wait() 方法返回，而返回的前提是该线程获取到了对象的锁                             |
| notifyAll()     | 通知所有等待在该对象上的线程                                                                                   |
| wait()          | 调用该方法的线程进入 WAITING 状态，只有等待另外线程的通知或被中断才会返回，需要注意，调用 wait() 方法后，会释放对象的锁 |
| wait(long)      | 超时等待一段时间，也就是等待长达 n 毫秒，如果没有通知就超时返回                                                    |
| wait(long, int) | 对于超时时间更细粒度的控制，可以达到纳秒                                                                        |

等待/通知机制，是指一个线程A 调用了对象O 的 wait() 方法进入等待状态，而另一个线程B 调用了对象O 的 notify() 或者 notifyAll() 方法，
线程A 收到通知后从对象O 的 wait() 方法返回，进而执行后续操作。上述两个线程通过对象O 来完成交互，而对象上的 wait() 和 notify()/notifyAll()
的关系就如同开关信号一样，用来完成等待方和通知方之间的交互工作。
```java
public class WaitNotify {
    static boolean flag = true;
    static Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        Thread waitThread = new Thread(new Wait(), "WaitThread");
        waitThread.start();
        Thread.sleep(1 * 1000);
        Thread notifyThread = new Thread(new Notify(), "NotifyThread");
        notifyThread.start();
    }

    static class Wait implements Runnable {

        @Override
        public void run() {
            // 加锁，拥有 lock 的 Monitor
            synchronized (lock) {
                // 当条件不满足时，继续 wait ，同时释放了 lock 的锁
                while (flag) {
                    try {
                        System.out.println("1");
                        System.out.println(Thread.currentThread() + " flag is true. wait " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                        lock.wait();
                        System.out.println("2");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                // 条件满足时，完成工作
                System.out.println(Thread.currentThread() + " flag is false. running " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
            }
        }
    }

    static class Notify implements Runnable {

        @Override
        public void run() {
            // 加锁，拥有 lock 的 Monitor
            synchronized (lock) {
                // 获取 lock 的锁，然后进行通知，通知时不会释放 lock 的锁
                // 直到当前线程释放了 lock 后， WaitThread 才能从 wait 方法中返回
                System.out.println(Thread.currentThread() + " hold lock. notify " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                lock.notifyAll();
                flag = false;
                try {
                    Thread.sleep(5 * 1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            // 再次加锁
            synchronized (lock) {
                System.out.println(Thread.currentThread() + " hold lock again. sleep " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                try {
                    Thread.sleep(5 * 1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

## 3.3 Thread.join() 的使用

如果一个线程A 执行了 thread.join() 语句，即：当前线程A 等待 thread 线程终止之后才从 thread.join() 返回。代码4-13如下：
```java
public class Join {

    public static void main(String[] args) {
        Thread previous = Thread.currentThread();
        for (int i = 0; i < 10; i++) {
            // 每个线程拥有前一个线程的引用
            Thread thread = new Thread(new Domino(previous), String.valueOf(i));
            thread.start();
            previous = thread;
        }
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + " terminate. ");
    }

    static class Domino implements Runnable {

        private Thread thread;

        public Domino(Thread thread) {
            this.thread = thread;
        }
        @Override
        public void run() {
            try {
                thread.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + " terminate. ");
        }
    }
}
```

Thread.join() 源码大概是这样的结构：
```java
// 加锁当前线程对象
public final synchronized void join() throws InterruptedException {
    // 条件不满足，继续等待
    while(isAlive()) {
        wait(0);
    }
    // 条件符合，方法返回
}
```
逻辑结构和等待/通知经典范式一致，即加锁、循环和处理逻辑。

## 3.4 ThreadLocal 的使用

```java
public class Profiler {
    // 第一次调用 get() 方法会进行初始化，每个线程只会执行一次
    private static final ThreadLocal<Long> TIME_threadLocal = new ThreadLocal() {
        @Override
        protected Object initialValue() {
            return System.currentTimeMillis();
        }
    };

    public static final void begin() {
        TIME_threadLocal.set(System.currentTimeMillis());
    }
    
    public static final long end() {
        return System.currentTimeMillis() - TIME_threadLocal.get();
    }
    
    public static void main(String[] args) throws InterruptedException {
        Profiler.begin();
        Thread.sleep(1000);
        System.out.println("Cost: " + Profiler.end() + " mills");
    }
}
```

# 4 线程应用实例

## 4.1 等待超时模式

调用一个方法时等待一段时间，如果该方法能够在给定的时间段之内得到结果，那么将结果立即返回，反之，超时返回默认结果。  
假设超时时间为 T ，那么在 now + T 之后就会超时。
```java
public synchronized Object get(long t) {
    long now = System.currentTimeMillis();
    long future = now + t;
    long remaining = t;
    while (result == null && remaining > 0) {
        wait(remaining);
        remaining = future - System.currentTimeMillis();
    }
    return result;
}
```

## 4.2 一个简单的数据库连接池示例

主干代码：
```java
public class ConnectionPool {
    private LinkedList<Connection> pool = new LinkedList<>();

    public ConnectionPool(int initialSize) {
        if (initialSize > 0) {
            for (int i = 0; i < initialSize; i++) {
                pool.addLast(ConnectionDriver.createConnection());
            }
        }
    }

    public void releaseConnection(Connection connection) {
        if (connection != null) {
            synchronized (pool) {
                pool.addLast(connection);
                pool.notifyAll();
            }
        }
    }

    public Connection fetchConnection(long mills) throws InterruptedException {
        synchronized (pool) {
            if (mills <= 0) {
                while (pool.isEmpty()) {
                    pool.wait();
                }
                return pool.removeFirst();
            } else {
                long future = System.currentTimeMillis() + mills;
                long remaining = mills;
                while (pool.isEmpty() && remaining > 0) {
                    pool.wait(remaining);
                    remaining = future - System.currentTimeMillis();
                }
                Connection result = null;
                if (!pool.isEmpty()) {
                    result = pool.removeFirst();
                }
                return result;
            }
        }
    }
}
```

## 4.3 线程池技术

主干代码：
```java
public class DefaultThreadPool<Job extends Runnable> implements ThreadPool<Job> {

    private static final int MAX_WORKER_NUMBERS = 10;
    private static final int DEFAULT_WORKER_NUMBERS = 5;
    private static final int MIN_WORK_NUMBERS = 1;

    // 任务列表
    private final LinkedList<Job> jobs = new LinkedList<>();

    // 工作者列表，即保存着所有的消费者列表
    private final List<Worker> workers = Collections.synchronizedList(new ArrayList<Worker>());

    // 工作线程的数量
    private int workerNum = DEFAULT_WORKER_NUMBERS;

    // 线程编号
    private int threadNum = 0;

    @Override
    public void execute(Job job) {
        if (job != null) {
            synchronized (jobs) {
                jobs.addLast(job);
                jobs.notify();
            }
        }
    }

    @Override
    public void shutdown() {
        for (Worker worker : workers) {
            worker.shutdown();
        }
    }

    @Override
    public void addWorkers(int num) {
        synchronized (jobs) {
            if (num + this.workerNum > MAX_WORKER_NUMBERS) {
                num = MAX_WORKER_NUMBERS - this.workerNum;
            }
            initializeWorkers(num);
            this.workerNum += num;
        }
    }

    /**
     *  移除工作线程，即移除消费者
     */
    @Override
    public void removeWorker(int num) {
        synchronized (jobs) {
            if (num >= this.workerNum) {
                throw new IllegalArgumentException("beyond workNum");
            }
            int count = 0;
            while (count < num) {
                Worker worker = workers.get(count);
                if (workers.remove(worker)) {
                    worker.shutdown();
                    count++;
                }
            }
            this.workerNum -= count;
        }
    }

    @Override
    public int getJobSize() {
        return jobs.size();
    }

    // 初始化工作者
    private void initializeWorkers(int num) {
        for (int i = 0; i < num; i++) {
            Worker worker = new Worker();
            workers.add(worker);
            Thread thread = new Thread(worker, "ThreadPool-Worker-" + threadNum++);
            thread.start();
        }
    }

    // 工作者，即消费者，负责消费任务
    class Worker implements Runnable {
        // 允许外界控制是否停止
        private volatile boolean running = true;

        @Override
        public void run() {
            while (running) {
                Job job = null;
                synchronized (jobs) {
                    while (jobs.isEmpty()) {
                        try {
                            jobs.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            Thread.currentThread().interrupt();
                            return;
                        }
                    }
                    // 取出一个Job
                    job = jobs.removeFirst();
                }
                if (job != null) {
                    job.run();
                }
            }
        }
        public void shutdown() {
            running = false;
        }
    }
}
```

## 4.4 小结

![][3]






[1]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/concurrent_art/4_1.png
[2]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/concurrent_art/4_2.png
[3]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/concurrent_art/4_3.png