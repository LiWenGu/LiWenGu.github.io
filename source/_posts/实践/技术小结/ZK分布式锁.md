---
title: ZK分布式锁
date: 2018-06-15 23:00:00
updated: 2018-06-18 21:43:00
tags:
  - 中间件
  - ZooKeeper
categories: 
  - 小结
  - ZooKeeper
comments: true
permalink: tech/zk_distribute_lock.html    
---

# 0  前言

 关于 ZooKeeper实现分布式锁，笔者在武汉小米一面（结果挂了）被问到过，因此记录如下。  
  
以下的理论知识源自`《 从Paxos到Zookeeper分布式一致性原理与实践 》`第六章，代码 完全根据书本理论进行实现，并且经多线程测试，在正常情况可行。  
  
源码：https://github.com/LiWenGu/MySourceCode/tree/master/example/src/main/java/com/lwg/zk_project

# 1 ZooKeeper实现排他锁

## 1.1 原理
  
核心点：  
1. `抢占式创建相同名称的临时节点`，谁成功创建节点，则代表谁获得了锁。
2. 没有创建成功该节点，并且该节点存在，则对该名称的节点进行删除监听。
3. 如果该节点被删除了，则继续重复第 1步。

## 1.2 流程图

原书流程图：  
![][1_0]  
我自己理解的流程：  
![][1_1]

## 1.3 代码实现

统一接口：
```java
/**
 * @Author liwenguang
 * @Date 2018/6/15 下午9:16
 * @Description
 */
public interface DistributedLock {

    /**
     * @Author liwenguang
     * @Date 2018/6/15 下午9:17
     * @Description 获取锁，默认等待时间
     */
    default void tryRead() throws ZkException { throw new RuntimeException("子类不支持"); }

    /**
     * @Author liwenguang
     * @Date 2018/6/15 下午9:18
     * @Description 获取锁，指定超时时间
     */
    default void tryRead(long time, TimeUnit unit) { throw new RuntimeException("子类不支持"); }

    void tryWrite() throws ZkException;

    default void tryWrite(long time, TimeUnit unit) { throw new RuntimeException("子类不支持"); }

    /**
     * @Author liwenguang
     * @Date 2018/6/15 下午9:18
     * @Description 释放锁
     */
    void release() throws ZkException;

}
```
  
核心代码：  
```java
private void tryGetLock() {
    CountDownLatch countDownLatch = new CountDownLatch(1);
    while (true) {
        try {
            zkClient.createEphemeral(EXCLUSIVE_LOCK_NAMESPACE + lockPath);
            log.info(Thread.currentThread().getName() + "获取锁成功");
            break;
        } catch (ZkNodeExistsException e) {
            // log.warn(Thread.currentThread().getName() + "获取锁失败");
            if (zkClient.exists(EXCLUSIVE_LOCK_NAMESPACE + lockPath)) {
                MyIZkDataListener myIZkChildListener = new MyIZkDataListener(countDownLatch);
                zkClient.subscribeDataChanges(EXCLUSIVE_LOCK_NAMESPACE + lockPath, myIZkChildListener);
            } else {
                countDownLatch.countDown();
            }
        }
        try {
            // 这里需要阻塞式通知，因此使用 countDownLatch实现
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    log.info("获取到了锁");
}

class MyIZkDataListener implements IZkDataListener {

    private CountDownLatch countDownLatch;


    public MyIZkDataListener(CountDownLatch countDownLatch) {
        this.countDownLatch = countDownLatch;
    }

    @Override
    public void handleDataChange(String dataPath, Object data) throws Exception { }

    @Override
    public void handleDataDeleted(String dataPath) throws Exception {
        //log.info(Thread.currentThread().getName() + "被回调了");
        zkClient.unsubscribeDataChanges(EXCLUSIVE_LOCK_NAMESPACE + lockPath, this);
        countDownLatch.countDown();
    }
}
```

# 2 ZooKeeper共享锁

## 2.1 原理
  
核心点：  
1. 无论是读请求（读锁）还是写请求（写锁）都进行创建`顺序`临时节点，只看后缀的数字我们可以理解为 一种从小到大的队列（例：我们在做订单请求的时候，对订单A做创建->  支付-> 完成三个操作，对应 ZK节点则节点A下有三个子节点，这时候节点A可以理解为一个队列）。
![][2_3]
2. 创建完成之后，对读锁，则判断该队列之前是否有写锁，如果有写锁，则对写锁做删除监听。对写锁，判断队列之前是否有锁，如果有锁，则对序号最大的锁做删除监听。
3. 删除监听触发，获取该锁节点下所有的子节点（一个节点即代表锁），重复第 2步。

## 2.2 流程图

原书流程图：  
![][2_0]  
我自己理解的流程：  
![][2_1]

## 2.3 代码实现

核心代码：  
```java
@Override
public void tryRead() throws ZkException {
    if (!zkClient.exists(SHARED_LOCK_NAMESPACE + lockPath)) {
        zkClient.createPersistent(SHARED_LOCK_NAMESPACE + lockPath);
    }
    CountDownLatch countDownLatch = new CountDownLatch(1);
    curNode = zkClient.createEphemeralSequential(SHARED_LOCK_NAMESPACE + lockPath + "/" + SHARED_READ_PRE, null);
    String curSequence = curNode.split(SHARED_READ_PRE)[1];
    log.info(curSequence + "创建读锁-R");
    while (true) {
        List<String> children = zkClient.getChildren(SHARED_LOCK_NAMESPACE + lockPath);
        // 记录序号比自己小的写请求
        List<String> writers = new ArrayList<>();
        for (String brother : children) {
            if (brother.startsWith(SHARED_WRITE_PRE)) {
                String sequence = brother.split(SHARED_WRITE_PRE)[1];
                if (curSequence.compareTo(sequence) > 0) {
                    writers.add(brother);
                }
            }
        }
        if (writers.isEmpty()) {
            // 没有比自己序号小的写请求，说明自己获取到了读锁
            //log.info(Thread.currentThread().getName() + "没有比自己序号小的写请求-R");
            break;
        } else {
            // 获取最近的那个写锁
            String lastWriter = SHARED_LOCK_NAMESPACE + lockPath + "/" + writers.get(writers.size() - 1);
            // 判断最近的那个写锁期间是否已经释放了
            if (zkClient.exists(lastWriter)) {
                MyReadIZkChildListener myReadIZkChildListener = new MyReadIZkChildListener(lastWriter, countDownLatch);
                zkClient.subscribeDataChanges(lastWriter, myReadIZkChildListener);
            } else {
                countDownLatch.countDown();
            }
        }
        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    log.info("获取到了锁-R");
}

class MyReadIZkChildListener implements IZkDataListener {

    private String lastWriter;
    private CountDownLatch countDownLatch;

    public MyReadIZkChildListener(String lastWriter, CountDownLatch countDownLatch) {
        this.lastWriter = lastWriter;
        this.countDownLatch = countDownLatch;
    }

    @Override
    public void handleDataChange(String dataPath, Object data) throws Exception { }

    @Override
    public void handleDataDeleted(String dataPath) throws Exception {
        //log.info(Thread.currentThread().getName() + "比自己序号小的那个写请求被释放了-R");
        zkClient.unsubscribeDataChanges(lastWriter, this);
        // 最近的那个写锁被释放了，但是不排除释放过程中，有其它写锁新加入，因此读锁需要重新获取列表
        countDownLatch.countDown();
    }
}

@Override
public void tryWrite() throws ZkException {
    CountDownLatch countDownLatch = new CountDownLatch(1);
    if (!zkClient.exists(SHARED_LOCK_NAMESPACE + lockPath)) {
        zkClient.createPersistent(SHARED_LOCK_NAMESPACE + lockPath);
    }
    curNode = zkClient.createEphemeralSequential(SHARED_LOCK_NAMESPACE + lockPath + "/" + SHARED_WRITE_PRE, null);
    String curSequence = curNode.split(SHARED_WRITE_PRE)[1];
    log.info(curSequence + "创建写锁-W");
    while (true) {
        List<String> children = zkClient.getChildren(SHARED_LOCK_NAMESPACE + lockPath);
        // 记录序号比自己小的请求
        List<String> writersOrReader = new ArrayList<>();
        for (String brother : children) {
            if (brother.equals(SHARED_WRITE_PRE + curSequence)) {
                // 排除自己
                continue;
            }
            String sequence = "";
            if (brother.contains(SHARED_WRITE_PRE)) {
                sequence = brother.split(SHARED_WRITE_PRE)[1];
            } else if (brother.contains(SHARED_READ_PRE)) {
                sequence = brother.split(SHARED_READ_PRE)[1];
            } else {
                // 异常名称节点的处理
            }
            if (curSequence.compareTo(sequence) > 0) {
                writersOrReader.add(brother);
            }
        }
        if (writersOrReader.isEmpty()) {
            // 没有比自己序号小的请求，说明自己获取到了读锁
            //log.info(Thread.currentThread().getName() + "没有比自己序号小的请求-W");
            break;
        } else {
            // 获取最近的那个锁
            String lastWriterOrReader = SHARED_LOCK_NAMESPACE + lockPath + "/" + writersOrReader.get(writersOrReader.size() - 1);
            // 判断最近的那个锁期间是否已经释放了
            if (zkClient.exists(lastWriterOrReader)) {
                MyWriteIZkChildListener myWriteIZkChildListener = new MyWriteIZkChildListener(lastWriterOrReader, countDownLatch);
                zkClient.subscribeDataChanges(lastWriterOrReader, myWriteIZkChildListener);
            } else {
                countDownLatch.countDown();
            }
        }
        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    log.info("获取到了锁-W");
}

class MyWriteIZkChildListener implements IZkDataListener {

    private String lastWriterOrReader;
    private CountDownLatch countDownLatch;

    public MyWriteIZkChildListener(String lastWriterOrReader, CountDownLatch countDownLatch) {
        this.lastWriterOrReader = lastWriterOrReader;
        this.countDownLatch = countDownLatch;
    }

    @Override
    public void handleDataChange(String dataPath, Object data) throws Exception {
    }

    @Override
    public void handleDataDeleted(String dataPath) throws Exception {
        //log.info(Thread.currentThread().getName() + "比自己序号小的那个请求被释放了，循环-W");
        zkClient.unsubscribeDataChanges(lastWriterOrReader, this);
        countDownLatch.countDown();
    }
}
```

# 3. 待续

readlock

[1_0]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/paxos2zookeeper/Paxos2zookeeper-6-1-16.png
[1_1]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/paxos2zookeeper/zk_lock_1.png
[2_0]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/paxos2zookeeper/Paxos2zookeeper-6-1-19.png
[2_1]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/paxos2zookeeper/zk_lock_2.png
[2_3]:http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/paxos2zookeeper/zk_shared_lock_1.png
