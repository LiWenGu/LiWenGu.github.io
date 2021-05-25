---
title: 对Redis集群与RedLock的理解
date: 2019-12-11 09:12:00
updated: 2019-12-11 09:12:00
tags:
  - redis
  - 分布式锁
categories: 
  - [技术总结, redis]
comments: true
permalink: tech/redis_cluster_and_redlock.html 
---

该文为回答该 Issue 的答案：https://github.com/Snailclimb/JavaGuide/issues/567


之前研究过一段时间，用的是 Redisson 框架，[这里是 Redission 实现 Redlock 的官方文档][0] 。以下为个人见解，可能有误

# 0. 背景

1. 平时理解的 Redis 的集群模式：http://www.redis.cn/topics/cluster-tutorial.html
2. Redlock 多节点：http://redis.cn/topics/distlock.html

重点如下：
1. 集群模式：Redis 集群是一个提供在多个Redis间节点间共享数据的程序集。

2. Redlock：有 N 个 Redis master。这些节点完全互相独立，不存在主从复制或者其他集群协调机制。
>这里 Redlock 的使用有四个要求，一是需要 N 个 Redis master，二是节点独立、三是不存在主从复制、四是不存在集群协调机制。后三个要求其实就说明了，你要用的这些 master 不能同时存在同一个集群中。

以下用图加深理解

# 1. 两主两从的集群与 Redlock

![][1]

红框代表的是一个标准的 Redis 集群：它是有六个 Redis 节点间共享数据的程序集。  
  
对于 Redlock 来说，这个集群中的两个 Master 不是互相独立的，而且是有集群协调机制的。因此它们不符合使用 Redlock 的场景。你会想，那我只连接一个 Master 可以吧，是可以的，但是也不是标准的 Redlock 的，因为不符合 N 个 Redis master 要求。

综上：如果你只有一个标准的集群，无论里面有多少个 Master，都是不符合 Redlock 的使用场景的，主要在于一个集群内的 Master 间它们不是互相独立的。

# 2. 两个独立集群下的 Redlock

![][2]

红框和蓝框分别代表两个独立的标准的 Redis 集群，它们分别是有三个 Redis 节点间共享数据的程序集。

对于 Redlock 来说，一共有两个符合要求的 Master，因此是符合 Redlock 的使用场景。这两个 Master 的同步只在自己的集群内和 Slave 同步。但是不允许这两个 Master 跨集群间同步，因为如果跨集群同步了，那么就不符合 Master 互相独立的要求了。

# 3. 复杂的两个独立集群下的 Redlock

![][3]

这里每个集群内部有两个 Master，此时对于 Redlock 来说，它只认为有一组 Master（Master1-Master2、Master1-Master22、Master11-Master2、Master11-Master22） 是互相独立的。（前提在于这两个集群之间并不同步）
>其实这里说的并不严谨。这里涉及 Redis 集群的哈希槽知识，对于客户端来说，连接 Redis 集群时，连接该集群下的任意一个节点即可，集群会自动给你对 key 做 hash，然后映射到不同的 Master 上，因此对于客户端来说，其实一个集群无论里面有多少个 Master 节点，客户端只会以为有一个节点。

# 4. 总结

1. Redlock 所说的多节点，指的是，独立的 Master 节点。但是集群模式下的 Master 它们是归属于同一个集群下的 Master 节点，它们互相是不独立的，所以这不是一回事。
2. 虽然 Redlock 所说的 Master 间不同步，但是为了高可用，这里每个 Master 节点都会有 Slave，用于冗余容灾，它们各自的 Master-Slave 是同步的。

# 5. 备注

关于图的连线，是直接连接 Master 还是连接集群，其实这里要说一下 Redis-cluster，客户端连接集群时，选择集群的任意一个节点连接，在集群中会自动转发到对应的 Master 做操作。  

---

**如果有错误请指正。**

[0]: https://github.com/redisson/redisson/wiki/8.-%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E5%92%8C%E5%90%8C%E6%AD%A5%E5%99%A8#84-%E7%BA%A2%E9%94%81redlock
[1]: https://user-images.githubusercontent.com/15909210/69705013-9e963280-112f-11ea-8d4d-2b2762f3fb54.png
[2]: https://user-images.githubusercontent.com/15909210/69705530-bcb06280-1130-11ea-9a72-d0e40303c59a.png
[3]: https://user-images.githubusercontent.com/15909210/69706179-1b2a1080-1132-11ea-8a24-705452579a8a.png
