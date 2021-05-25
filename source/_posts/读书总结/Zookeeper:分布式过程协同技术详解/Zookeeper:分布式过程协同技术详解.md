---
title: 《Zookeeper:分布式过程协同技术详解》阅读总结
date: 2020-01-23 17:21:00
updated: 2020-01-23 17:21:00
comments: true
tags:
  - zookeeper
categories: 
  - [读书总结, Zookeeper:分布式过程协同技术详解]
permalink: zookeeper_distributed_process_coordination/1.html    
---

# 0. 背景

公司从 naocs 的注册中心到 zookeeper 注册中心转变，自己也好久没看过 zookeeper 相关了，上次应该看的是《从Paxos到Zookeeper》一书，这次正好趁年假把另一本经典的书《Zookeeper:分布式过程协同技术详解》看下。

# 1. 总览

一共十章，八章偏入门实战，敲代码和实战理论为主，在 github 上也有相应的源码。第九章是原理讲解，最后一章是配置详解。  
前八章应该属于入门实战，通过介绍四大节点类型和监听，来实现一个简单的分布式任务协调中心。  
第九章通过对源码讲解，对关键的“顺序一致性”、“事务”做代码上的了解。  
最后一章应该属于参数调优，适合运维和部署者，属于优化点。

# 2. 入门实战

入门实战分布式任务调度中心书中源码地址：https://github.com/fpj/zookeeper-book-example  

关键点：  
1. 临时节点
2. 临时有序节点
3. 永久节点
4. 永久临时节点
5. watch 监听
6. 异步调用
7. ACL 权限控制
8. watch 羊群效应的避免
9. watch 事件类型业务处理
10. curator 开源客户端的使用

> 观察者的横向扩展用于读性能提高。而在选举时的跟随者以及领导者时的 zxid 的 zab 广播协议过程是通过延长领导者选举时间、超过半数来避免脑裂的产生。

# 3. 源码分析

关键步骤：
1. git clone https://github.com/apache/zookeeper.git
2. ant eclipse 编译
3. idea 导入: file - new - Project From Existing Sources
4. zookeeper-jute 项目需要编译得到生成之后的代码: maven clean install
5. 复制 zookeeper-jute 项目的 target/classed/org 到 zookeeper-server 下
6. 在 zookeeper-server 项目的 org.apache.zookeeper.version 包新建 Info 类：
```java
public interface Info {
    int MAJOR=3;
    int MINOR=5;
    int MICRO=6;
    String QUALIFIER=null;
    String REVISION_HASH="c11b7e26bc554b8523dc929761dd28808913f091";
    String BUILD_DATE="01/19/2020 10:13 GMT";
}
```
7. 启动 server：执行 org.apache.zookeeper.server.quorum.QuorumPeerMain 类并带 JVM 参数：-Dlog4j.configuration=file:/Users/liwenguang/SourceCode/middle/zookeeper/conf/log4j.properties 以及启动参数：conf/zoo_sample.cfg

>-Dzookeeper.4lw.commands.whitelist=* 用于 telnet 命令查询。

8. 至此，server 启动完毕
9. 启动 client：执行 org.apache.zookeeper.ZooKeeperMain，启动参数：-server 127.0.0.1:2181 ls /zookeeper

>- ls /zookeeper 为命令

10. 至此 client 启动测试完毕

# 4. 最后

翻译的好烂！错别字不说，句子翻译的都不通畅！但是真适合入门（第九章除外），第九章适合看了《从Paxos到Zookeeper》之后再看。