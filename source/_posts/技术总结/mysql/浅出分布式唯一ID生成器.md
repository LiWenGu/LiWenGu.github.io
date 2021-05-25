---
title: 浅出分布式唯一ID生成器
date: 2019-09-05 23:55:00
updated: 2019-09-05 23:55:00
tags:
  - 分布式
  - 唯一ID生成算法
categories: 
  - [技术总结, mysql]
comments: true
permalink: tech/unique_id_1.html 
---

# 0. 背景

近日由于订单量+机器数量增加，导致原来使用时间戳生成唯一订单号的方法行不通了，出现很多主键冲突      
并且由于主键冲突导致的事务死锁的几率也随之加大，进而需要补的数据越来越多！因此急需一个全局唯一ID生成方式。  
但由于现在单体框架还在拆解过程中，新的生成方式需要兼容单体应用（多台集群）  
最好能用最简单的方式先解决，待以后服务拆解出来后考虑可用性扩展性。

<!-- more -->

# 1. 调研

大概有以下几个选择，优先级由高到低：  
1. snowflake
2. 数据库唯一ID主键约束
3. redis 原子递增
4. UUID
5. leaf
6. uid-generator

优先级 snowflake 主要是有以下理由：  
1. snowflake 现阶段用一个工具类就能直接上手使用
2. 现在分库分表也使用了 snowflake，以后可以做统一的唯一ID服务
3. 很多分布式ID中间件都基于 snowflake，以后也方便扩展，而不会出现**由于算法不同，新算法生成的序列号可能被以前已经生成过**

# 2. 实际使用

1. 开始考虑使用IP的最后一位作为 workid，因为已经和运维确认过，现在线上机器是最后是连号，不会重复，但是因为默认 workid 最大是 32，就没有将 datacenterid 都改为 workid，因为我发现我们 hostname 是带编号的，而且是递增的规律：xxservice001，最后三位是序号，因此直接取最后三位数字，作为 workid。

>snowflake 默认使用 `1+41+10+12` ID组装模式，其中 10 bit 是 datacenter + workid

# 2.1 时钟回拨问题

关于时钟回拨，一般解决方法有：
1. 机器之间定时同步时间
2. 通过请求其它机器获取平均时间戳对比（Leaf）
3. 在阈值内不断获取当前时间直到大于上次申请时间
4. 使用第三方存储号段来解决（我个人倾向号段方案）

由于正面临拆分，以后会有专门的 id 生成服务，因此现在并没有在单体服务上做时钟回拨的解决，我选择最简单的 hutool 工具引用即可，但是它仍可能有一定几率有时钟回拨。
>现阶段运行了两周，还没发现这个错误。

# 3. 其它ID算法简析

稍微系统的学习一下业内的分布式ID生成方案

## 3.1 uid-generator

以中间件的形式提供服务，基于 snowflake 的优化版，使用双 buffer 环（我个人理解为生产者和消费者模式），一个 buffer 生产号段，一个 buffer 消费号段。

snowflake 各个分配可自定义：  
- 1bit：第一位不用，用于标志 0 ，表示正数
- 28bits：每个机器当前时间，单位秒
- 22bits：机器数量
- 13bits：每秒的并发数

1. 默认是秒级并发，常规的 `snowflake` 算法是基于毫秒级(41 bits)，但是没有必要这么精确，每台机器每秒并发最多8192够用了。    
2. 接着是环形数组，通俗来讲就是生产者和消费者模型，一个不能生产过快，一个不能消费过快。同时该模型又有扩容机制，即生产者到一定量后扩容。生产者每次生产者的数量。由于是周期性填充生产并缓存，因此可以接受短时间内第三方存储服务不可用。  
3. 默认支持通过 mysql 存储机器申请情况，这可以用其它来代替，例如 zk，但是这需要**我依赖的需要高可用** 

>由此可见, 不管如何配置, CachedUidGenerator总能提供600万/s的稳定吞吐量, 只是使用年限会有所减少. 这真的是太棒了
>https://github.com/baidu/uid-generator/blob/master/README.zh_cn.md

## 3.2 leaf

也是以中间件的形式提供服务。

>目前Leaf的性能在4C8G的机器上QPS能压测到近5w/s，TP999 1ms，已经能够满足大部分的业务的需求

有两种生产号段方式：`Leaf-segment`、`Leaf-snowflake`

### 3.2.1 Leaf-segment

基于 mysql 数据库的号段模式，不是每一次都访问数据库，而是在数据库记录哪台机器分配了哪个号段区间。  
  
在每次消费时，当前机器访问 Leaf 时，Leaf 会得到该机器是否已经发了号段，并判断下一个号段的状态，接着从数据库中更新号段信息，并将号段到载入内存中，防止出现当前号段消费完后，加载下一个号段时的卡顿。 
  
由此得知可以短暂的容忍 mysql 一段时间内不可用。

### 3.2.2 Leaf-snowflake

标准的 `1+41+10+12` ID组装模式，默认使用 zk 做 leaf 集群多机房部署以达到高可用，主要是利用临时节点，记录当前正在访问 leaf 的消费端。
>解决了时钟回拨问题：3s一次上传 leaf 机器节点当前的时间，并进行 RPC 调用其它机器的时间，得到平均时间后，有个阈值（考虑网络延迟）进而通过判断当前机器时间与平均时间，来解决时钟回拨的问题。


## 3.3 UUID

唯一，但是无序，且长度太长，一般来说，数据库存都是 bigint(20) 即 64 bit，对应在 java 中使用 long，也是 64bit，而 uuid 是 128 bit，且是字母数字组合，需要 char(128)，空间浪费很严重

## 3.4 redis

基于 Redis 全局递增，步长设置。  
使用 lua 脚本结合 snowflake 来实现   

例如有三台业务机器，两台 redis 作为id生成机器：`1+41+10+12` 模式，其中的 workid 使用 redis 集群中的机器编号，这样就会请求的每个 redis 获取的 redis 都不会重复，但是仍旧有时钟回拨的问题。

## 3.5 idx_mysql_id

基于 mysql 的主键，强依赖mysql，而且性能低，适合前期使用，并发量大时，不仅每次要访问数据库，而且还要做保证数据库的高可用

## 3.6 snowflake

基本上现在主流的都会或多或少参考该算法
一共64bit，
1. 1bit默认为0，表示正数
2. 41bit作为时间戳毫秒值
3. 10bit作为工作机器
4. 12bit作为每个机器每毫秒下最多能支持多少个并发请求生成号  

cn.hutool.core.lang.Snowflake
```java
// 获取当前时间，hutool 内部有个定时任务刷时间
long timestamp = genTime();
// 将当前时间与上次申请时间对比，判断是否出现时钟回拨
if (timestamp < lastTimestamp) {
     //如果服务器时间有问题(时钟回拨) 报错。
    throw new IllegalStateException(StrUtil.format("Clock moved backwards. Refusing to generate id for {}ms", lastTimestamp - timestamp));
}
// 微秒下的并发走这里
if (lastTimestamp == timestamp) {
    // 微秒下进行序号 +1，默认最多支持微秒 8192 并发量
    sequence = (sequence + 1) & sequenceMask;
    if (sequence == 0) {
        timestamp = tilNextMillis(lastTimestamp);
    }
} else {
    // 并发数重置
    sequence = 0L;
}
// 重置上一次时间
lastTimestamp = timestamp;
```

# 4. 个人认为好的解决方案

## 4.1 号段

记录请求的机器ip+port（能唯一定位该机器的标识），然后分配号段，例如1000~3000，另一个机器过来请求，分配另一个号段，例如 4000~9000，这样。主要是依赖第三方存储号段分配情况，例如使用数据库，Redis，Zookeeper 都可以。没有

## 4.2 算法生成

就是 snowflake，不同在于根据业务的不同，可以对其中的 bit 做不同的分配，例如毫秒位改为秒位，这样多了三个位可以放给机器位，适合并发量不大，但是机器多的场景，或者机器位给最后的并发请求量位，适合并发量大，机器数量比 1024 少的场景。  
  
而且这个也可以进化为号段，但是会有很多永远都不会用到的号段，需要有个重复利用的方案。但是这解决了普通号段模式的订单有序性问题。

# 5. 参考

分布是唯一ID系列5篇：https://www.cnblogs.com/itqiankun/p/11350857.html
美团分布式ID系列2篇：https://tech.meituan.com/2019/03/07/open-source-project-leaf.html
多key的预备：https://mp.weixin.qq.com/s/PCzRAZa9n4aJwHOX-kAhtA?