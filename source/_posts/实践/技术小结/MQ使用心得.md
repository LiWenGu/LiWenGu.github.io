---
title: MQ使用心得
date: 2020-03-31 22:25:00
updated: 2020-03-31 22:25:00
tags:
  - MQ
  - 中间件
categories: 
  - 实践
  - 技术小结
comments: true
permalink: tech/mq_use.html    
---

![][0]

<!--more-->

# 1. 前提

消息中间件服务器基于阿里云 ONS（商业版RokcetMQ）  
消息 SDK 代码基于：`ons-client-ext 1.8.4-final` 版本

## 1.1 名词

1. Group：组，代表具有相同角色的生产者组合或消费者组合，称为生产者组或消费者组，例如一个支付系统，里面有10台机器，这10台就是一个组
2. Topic：一个 Topic 标识为一类消息类型，比如交易类信息、支付类消息、积分类消息。
3. Tag：一个 Tag 标识为一类消息中的二级分类，比如交易类信息下的交易创建、交易完成。

## 1.2 为什么引入消息队列

主要两个原因：
1. 解耦：当某个用户支付完某个订单，接着第三方支付成功回调到支付系统，此时，支付系统需要通知到订单系统、以及积分系统。此时用消息中间件将支付系统和其它系统解耦。
2. 削峰填谷：当某个活动高峰期，大量用户支付，每个用户支付完后积分系统都需要更新每个用户的积分。此时更新用户的积分可以做延迟顺序消息，做到高峰期的流量平滑。

## 1.3 使用消息队列需要注意什么

1. 首先考虑是否真的需要引入消息队列？
2. 这个业务是使用普通消息、顺序消息还是延迟消息还是事务消息？
3. 该如何正确的选择 Topic 和 Tag？
>以天猫交易平台为例，订单消息和支付消息属于不同业务类型的消息，分别创建 Topic_Order 和 Topic_Pay，其中订单消息根据商品品类以不同的 Tag 再进行细分，列如电器类、男装类、女装类、化妆品类等被各个不同的系统所接收。
>通过合理的使用 Topic 和 Tag，可以让业务结构清晰，更可以提高效率。

4. 时刻注意订阅关系一致的问题
>订阅关系一致指的是同一个消费者 Group ID 下所有 Consumer 实例的处理逻辑必须完全一致。一旦订阅关系不一致，消息消费的逻辑就会混乱，甚至导致消息丢失。

即同一个 Group ID 下的所有机器，订阅的 Topic 和 Tag 必须完全一样  
注意：
1. 不要设置错了 Group ID，这样会导致正确的 Group 出现消息不一致
2. Topic 或 Tag 不要修改或删除，如果必须要这样，做好消息丢失的可能
3. 每次对的消息的订阅信息做了操作时，在消息队列控制台查看订阅关系是否一致

# 2. 启动配置

生产者配置对应表：红色为必填，紫色建议根据业务调整
![][1]

消费者线程源码：
1. 无界队列：理论上消费消息足够多，会出现 OOM，因此建议做限流处理
2. ConsumeThreadMin 默认为 20，可以根据机器和业务特性进行调整：
ConsumeMessageOrderlyService.java
```java
this.consumeRequestQueue = new LinkedBlockingQueue<Runnable>();
this.consumeExecutor = new ThreadPoolExecutor(
    this.defaultMQPushConsumer.getConsumeThreadMin(),
    this.defaultMQPushConsumer.getConsumeThreadMax(),
    1000 * 60,
    TimeUnit.MILLISECONDS,
    this.consumeRequestQueue,
    new ThreadFactoryImpl("ConsumeMessageThread_"));
```

# 3. 使用消息队列时可能碰到的问题

## 3.1 发送时失败

1. 在 `ons-client 1.7.9` 版本经常会碰到，但是在 `ons-client 1.8.4` 版本基本碰到不到
2. `ons-client 1.8.4` 版本消息发送失败时会自动重试，因此当代码编写正确时（配置正确），但是仍然出现发送失败时，最大可能是 ONS 服务的问题异或是机器网络延迟问题

## 3.2 消息重复接收

1. 虽然 ONS 提供 Exactly-Once 特性来保证消息的最终处理结果写入到数据库有且仅有一次。但是实际中，即时开启了该特性，仍然有极小概率出现重复写入，而且随着分布式机器的增加，几率也会增加（阿里云官方人员也证实确实会这样）。  
2. 虽然最佳方案是消息接收时业务处理保证幂等，但是业务各式各样，尤其是复杂业务时，很难保证幂等。因此选择另一个和业务无关方案：通过 Redission 锁 + MessageKey 来保证同一个消息内容在某个时间段内仅消费一次。
3. 此方案强依赖 Redis，因此需要保证 Redis 集群的可用性。

1. 消息发送时，使用消息内容的 MD5 作为 MessageKey，由于 md5 会出现不同内容得到相同的 md5，因此额外加上 messageTag（也可以使用其它信息摘要算法）
2. 消息接收时，通过 Redission 分布式锁对其 MessageKey 做锁（可以使用其它分布式锁）
```java
// 消息发送
Message message = new Message();
message.setKey(messageTag + "_" + DigestUtil.md5Hex(msgContent));
send(message);

// 消息接收
boolean tryLockSuc = RedissionLock.tryLock(messageKey, timeout);
if (tryLockSuc) {
    // 成功获得锁，进行业务处理
} else {
    // 失败，说明该消息内容曾经处理过，此时是重复消息，直接返回
    return;
}
```

## 3.3 消息消费到一半机器异常宕机

1. 主要通过 Redis 来保证消息消费的完整性，注意设置 TTL。
2. 枚举表示：-1 表示消息消费异常，1 表示消息正常消费
```java
String msgSchedule = redisCloudService.get(messageKey);
msgSchedule = redisCloudService.get(messageKey);
if (msgSchedule == null || -1 == msgSchedule) {
    // 消息从未消费过，或上次异常消费，此时需要重新消费
    // 根据业务特性，是否需要重新处理或不处理...完
    redisCloudService.set(messageKey, 1);
    return;
} else if (1 == msgSchedule) {
    // 消息上次已经消费完成
    return;
}
```

## 3.4 消息丢失

消息丢失最大的可能就是消息不一致造成的
1. 消息不一致的原因无非三种：Group ID 设置错误，Topic 设置错误，Tag 设置错误，逐一排查即可
2. 解决完消息不一致后，定位消息不一致发生的时间段，将消息消费点重置当这个时间段开始，进行消息重新消费
3. 由于我们使用了【3.2消息重复接收】的方案，因此不会出现已经消费的消息重复消费

# 4. 更多

1. 消息队列不是解决某个问题的银弹。如果不熟悉它，只会引发更多的问题。  
2. 在 `MQVersion` 类中可以看到 `CURRENT_VERSION` 字段是 4.3.6，猜测 ONS 是基于 RocketMQ 4.3.6 版本，因此可以可以了解 RocketMQ 4.3.6 版本。
3. 推荐书籍《RocketMQ技术内幕》
4. B站 RocketMQ 官方视频：https://space.bilibili.com/271666652?from=search&seid=18398985987289994532
5. ONS 作者讲解 ONS 原理视频：https://v.youku.com/v_show/id_XODY4ODE3OTY0.html?from=s1.8-1-1.2

---

本文尽量为提及的每个数据和说法都列出可靠、可考的佐证，但由于作者能力有限，水平有限，难免有所疏漏，欢迎指出并讨论

[1]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/somephoto/mq_how.png