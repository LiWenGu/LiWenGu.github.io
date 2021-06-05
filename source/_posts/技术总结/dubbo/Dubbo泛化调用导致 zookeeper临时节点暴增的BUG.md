---
title: Dubbo泛化调用导致 zookeeper临时节点暴增的BUG
date: 2021-05-14 19:30:00
updated: 2021-05-14 19:30:00
tags:
  - dubbo
categories: 
  - [技术总结, dubbo]
comments: true
permalink: tech/dubbo/$invoke_zknode_problem.html    
---

# 0. 背景

dubbo 2.7.5，最近发现 zk 的 node 越来越多，监控报警半夜给我打电话。后来排查发现是业务方使用泛化调用不规范！
但是排查过程学习到了很多。

<!--more-->

# 1. 监控

线上 zk 监控主要是监控 zk 的 node 数量，以及机器 cpu memory，报警主要是 zk 的 node 数量报警，到了阈值。

# 2. 排查

先查看在报警前后的 zk snapshot 文件有什么变化，将 zk 快照文件转译为文本：
java -cp ../../zookeeper-3.4.8.jar:../../lib/slf4j-api-1.6.1.jar org.apache.zookeeper.server.SnapshotFormatter snapshot.b0008ebdd > 05141102.txt
这样你就得到了两份 snapshot，一份是报警前一段时间，正常的快照 node，一份是报警后的快照 node。

# 3. Dubbo 节点规则

/dubbo/{com.xx.Service}/consumers/consumer%3A%2F%2F{ip.ip.ip.ip}%2Forg.apache.dubbo.rpc.service.GenericService%3Fapplication%3D{app-name}%26category%3Dconsumers%26check%3Dfalse%26dubbo%3D2.0.2%26generic%3Dtrue%26interface%3D{com.xx.Service}%26lazy%3Dfalse%26loadbalance%3Drandom%26pid%3D26879%26release%3D2.7.3.5-ext%26side%3Dconsumer%26sticky%3Dfalse%26timestamp%3D1620959181596
/dubbo/服务提供者接口/consumers/泛化接口?timestamp=
通过搜索 zk 报警之后的快照文件发现有接近3000个泛化调用统一接口
接口查看该 node 的 ephemeralOwner，定位到该业务服务，同时查看线上该业务服务的发布记录，发现该业务服务的线上发布时间和故障报警时间重合：当该服务重启发布的时候，报警消失，猜测应该是因为机器重启，临时泛化节点都下掉。后来定位代码，发现泛化调用后没有显示的调用 .destory() 销毁泛化调用的 node！

# 4. 解决

1. 泛化调用后，finally 块代码加入 reference.destory() 方法。
2. 通过内存缓存 GenericService genericService = reference.get() 获得的 GenericService 类。
3. 不用泛化调用，直接调用。
