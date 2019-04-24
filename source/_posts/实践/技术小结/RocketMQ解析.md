---
title: RocketMQ解析
date: 2019-04-24 00:23:00
updated: 2019-04-24 00:23:00
tags:
  - RocketMQ
  - 中间件
categories: 
  - 小结
  - MQ
comments: true
permalink: tech/RocketMQ_jiexi.html    
---

# 1 前置知识

可能会有问题，如果描述不当请指出，因为自己对 Netty 源码没看多少

## 1.1 Netty 为什么快？  

自制协议，减少传输流大小。  
Netty 使用 Reactor 线程模型，发挥 epoll 优点  
使用 DirectBuffer ，并池化，加快数据分配效率  
局部串行无锁化，具体为 handler 没有线程转换（这里我有点不懂）

## 1.2 DirectBuffer、MappedByteBuffer 为什么快？  

## 1.3 RocketMQ 不使用 ZooKeeper 作为注册中心的原因，以及自制的 NameServer 优缺点？

ZooKeeper 作为支持顺序一致性的中间件，在某些情况下，它为了满足一致性，会丢失一定时间内的可用性，RocketMQ 需要注册中心只是为了发现组件地址，在某些情况下，RocketMQ 的注册中心可以出现数据不一致性，这也是 NameServer 的缺点，因为 NameServer 集群间互不通信，它们之间的注册信息可能会不一致，而且有新的服务器，NameServer 并不会立马通知到 Produer，而是由 Produer 定时去请求 NameServer 获取最新的 Broker/Consumer 信息，但是 NameServer 实现更加简单逻辑更少

# 2 单机消息中心

一个消息中心，最基本的需要支持多生产者、多消费者，例如下：

```java
class Scratch {

    public static void main(String[] args) {
        // 实际中会有 nameserver 服务来找到 broker 具体位置以及 broker 主从信息
        Broker broker = new Broker();
        Producer producer1 = new Producer();
        producer1.connectBroker(broker);
        Producer producer2 = new Producer();
        producer2.connectBroker(broker);

        Consumer consumer1 = new Consumer();
        consumer1.connectBroker(broker);
        Consumer consumer2 = new Consumer();
        consumer2.connectBroker(broker);

        for (int i = 0; i < 2; i++) {
            producer1.asyncSendMsg("producer1 send msg" + i);
            producer2.asyncSendMsg("producer2 send msg" + i);
        }
        System.out.println("broker has msg:" + broker.getAllMagByDisk());

        for (int i = 0; i < 1; i++) {
            System.out.println("consumer1 consume msg：" + consumer1.syncPullMsg());
        }
        for (int i = 0; i < 3; i++) {
            System.out.println("consumer2 consume msg：" + consumer2.syncPullMsg());
        }
    }

}

class Producer {

    private Broker broker;

    public void connectBroker(Broker broker) {
        this.broker = broker;
    }

    public void asyncSendMsg(String msg) {
        if (broker == null) {
            throw new RuntimeException("please connect broker first");
        }
        new Thread(() -> {
            broker.sendMsg(msg);
        }).start();
    }
}

class Consumer {
    private Broker broker;

    public void connectBroker(Broker broker) {
        this.broker = broker;
    }

    public String syncPullMsg() {
        return broker.getMsg();
    }

}

class Broker {

    private LinkedBlockingQueue<String> queue = new LinkedBlockingQueue(Integer.MAX_VALUE);

    // 实际发送消息到 broker 服务器使用 Netty 发送
    public void sendMsg(String msg) {
        try {
            queue.put(msg);
            // 实际会同步或异步落盘，异步落盘使用的定时任务定时扫描落盘
        } catch (InterruptedException e) {

        }
    }

    public String getMsg() {
        try {
            return queue.take();
        } catch (InterruptedException e) {

        }
        return null;
    }

    public String getAllMagByDisk() {
        StringBuilder sb = new StringBuilder("\n");
        queue.iterator().forEachRemaining((msg) -> {
            sb.append(msg + "\n");
        });
        return sb.toString();
    }
}
```

1. 没有实现真正执行消息存储落盘
2. 没有实现 NameServer 去作为注册中心，因为单机版在同一个 JVM 中  
3. 使用 LinkedBlockingQueue 作为消息队列，注意，参数是无限大，在真正 RocketMQ 也是如此是无限大，理论上内存数据抛弃的问题，但是会有内存泄漏问题（阿里巴巴开发手册也因为这个问题，建议我们使用自制线程池）。  
4. 没有使用多个队列（即多个 LinkedBlockingQueue）来模拟顺序消息，RocketMQ 的顺序消息是通过生产者和消费者同时使用同一个 MessageQueue 来实现
5. 没有使用 MappedByteBuffer 来实现文件映射从而使消息数据落盘非常的快（实际 RocketMQ 使用的是 FileChannel+DirectBuffer）

# 3 分布式消息中心

## 3.1 问题与解决

1. 消息丢失的问题
当你系统需要保证百分百消息不丢失，你可以使用生产者每发送一个消息，消息中间件返回一个消息成功的反馈消息，在 RocketMQ 是 Broker Master 来执行，即每发送一个消息，同步落盘后才返回生产者消息发送成功，这样只要生产者得到了消息发送生成的返回，事后除了硬盘损坏，都可以保证不会消息丢失，但是这同时引入了一个问题，同步落盘怎么才能快？
2. 同步落盘怎么才能快？
使用 FileChannel + DirectBuffer 池，使用堆外内存，加快内存拷贝  
使用数据和索引分离，当消息需要写入时，使用 commitlog 文件顺序写，当需要定位某个消息时，查询 index 文件来定位，从而减少文件IO随机读写的性能损耗
3. 消息堆积的问题？
后台定时任务每隔72小时，删除旧的没有使用过的消息信息  
根据不同的业务实现不同的丢弃任务，具体参考线程池的 AbortPolicy，例如FIFO/LRU等（RocketMQ没有此策略）  
消息定时转移，或者对某些重要的 TAG 型（支付型）消息真正落库
4. 定时消息的实现？
实际 RocketMQ 没有实现任意精度的定时消息，因为它实现定时消息的原理是，创建特定时间精度的 MessageQueue，例如生产者需要定时1s之后被消费者消费，你只需要将此消息发送到特定的 MessageQueue，然后消费者消费此消息，使用 newSingleThreadScheduledExecutor 实现
5. 顺序消息的实现？
与定时消息同原理，特定的 MessageQueue 即可
6. 分布式消息的实现？
RocketMQ4.3 起支持，原理为2PC，即两阶段提交，prepared->commit/rollback：生产者发送事务消息Topic1，RocketMQ 首先更改该消息的 Topic 为 Topic-Prepared，然后定时回调生产者的本地事务A执行状态，根据本地事务A执行状态，是否会将该消息置为 Topic1-Commit 或 Topic1-Rollback，消费者就可以正常找到该事务消息或者不执行等。
7. 消息的 push 实现？
注意，RocketMQ 以及说自己会有低延迟问题，其中就包括这个消息的 push，因为这并不是真正的消息主动的推送到消费者，而是 Broker 定时任务每5s推送消息到消费者，从而实现的消息推送
8. 消息重复发送的避免？
RocketMQ 会出现消息重复发送的问题，因为在网络延迟的情况下，这种问题可能会发送，避免会使程序复杂度加大，因此默认允许消息重复发送。让使用者在消费者端去解决该问题，即需要消费者端支持幂等性消费，最简单的解决方案是消费记录有个消费状态字段，或者使用一个集中式的表，来存储消息的消息状态，具体实现可以查询关于消息幂等消费的解决方案
9. 其它
包括组件通信间使用 Netty 的自定义协议。  
广播消费/集群消费。  
消息过滤器（在 Broker 端查询消息文件时就使用过滤服务器进行过滤）  
生产者、消费者群组的概念，这是为了支持集群消费以及横向扩展，具体看官网就能大概了解。

# 4 参考

1. 关于 RocketMQ 对 MappedByteBuffer 的一点优化：https://lishoubo.github.io/2017/09/27/MappedByteBuffer%E7%9A%84%E4%B8%80%E7%82%B9%E4%BC%98%E5%8C%96/
2. 阿里中间件团队博客-十分钟入门RocketMQ：http://jm.taobao.org/2017/01/12/rocketmq-quick-start-in-10-minutes/
3. 分布式事务的种类以及 RocketMQ 支持的分布式消息：https://www.infoq.cn/article/2018/08/rocketmq-4.3-release