---
title: Nacos源码1_配置管理源码
date: 2019-08-03 18:50:10
updated: 2019-08-03 18:50:10
tags:
  - 中间件
  - Nacos
categories: 
  - 实践
  - 技术小结
comments: true
permalink: tech/nacos1_config1.html    
---

# 1. 本地编译

1. fork 源地址：https://github.com/alibaba/nacos.git
2. git clone 自己 fork 版本
3. 现在基于 1.1.3 版本，nacos 项目基于标签做的版本发布，因此直接 git checkout 1.1.3 然后再基于此新建一个分支，用于源码的修改等
4. 在根目录下运行：mvn clean -Dmaven.test.skip=true，下载相关依赖包，检查环境
5. 运行 console 项目主类，加上 VM options：-Dnacos.standalone=true  
![][1]

6. 我们可以在控制台查看 banner 的相关运行信息

# 2. 修改字符分隔符方便调试

因为报文，使用了特殊的字符，不方便 http 测试请求，因此修改 com.alibaba.nacos.config.server.utils 包下的 MD5Util 类：  
```java
WORD_SEPARATOR_CHAR = '~'
LINE_SEPARATOR_CHAR = '*'
```

# 3. 发布订阅者模式

配置管理的实现，使用了大量的 ScheduledExecutorService 以及事件发布订阅者模式

1. 事件分发器/发布者：com.alibaba.nacos.config.server.utils.event.EventDispatcher
2. 两个监听者：  
AsyncNotifyService 监听 ConfigDataChangeEvent 事件，当配置做更新操作时，进行同步并异步做 http 请求到其它的健康节点
LongPollingService 监听 LocalDataChangeEvent 事件，当配置做更新操作时，一是更新 CacheItem（内存级），二是对当前监听的客户端们做响应
3. 两种事件：
ConfigDataChangeEvent：需要节点间同步配置改动的事件
LocalDataChangeEvent：本节点配置改动的事件

# 4. 数据库访问数据

单机版使用 derby 嵌入式数据库，只能由一个进程访问，查看数据需要停止 Nacos 进程才行，稍微麻烦点，不过 idea 支持 derby 数据查看，会方便点：  

![][2]

数据库的具体信息，从 LocalDataSourceServiceImpl 可以查看，包括数据存放地址等，这是可以通过全局配置来设置的

url：/Users/{user.home}/nacos/data/derby-data  
username：nacos  
password：空  

后台 console 的账号和密码根据配置文件可以查看：console/src/main/resources/META-INF/schema.sql，如果想修改，因为还有一个 Role 表，需要改动 role 和 user 表

# 5. 配置接口

## 5.1 配置的监听

客户端监听注册中心的配置，其实本质是根据 groupKey（AppId+groupId+namespace）获取对应的 md5，即配置内容的 md5 进行对比是否有更新。默认等待 30s，即是否在 30s 内有改动。

配置的改动监听有三种场景：A 表示客户端，B表示注册中心  
1. A->B，其实A的配置很老了，B的配置是最新的，B的监听接口会先根据本节点的 CacheItem 内存的值进行匹配，如果匹配失败，说明A 的值是旧的，会立刻返回给 A groupKey（不是md5或内容），让A去查询最新的配置值（多此一举么~）
2. A->B，其实A的配置就是最新的，B的配置和A的配置一样，那么 A 请求阻塞 30s（默认Long-Pulling-Timeout配置），B的监听接口使用 Servlet 3.0 异步进行 10s 一次的循环，来根据 groupKey 拿配置的md5，进行对比是否改动，如果改动则返回，不改动理论上会等待30s。
3. A->B，属于情况2的变种，就是，在情况2等待30s过程中，B2配置改变了，那么会根据事件分发以及 http 内部节点之间请求，B2会通知B，B最后来通知A，此时A会立刻返回

## 5.2 配置的修改

post/put/update/delete 等情况，这里有两个要注意，一是自己节点的内存更新，和其它节点的通知更新

1. 自己节点：先存数据库 derby，然后做事件 ConfigDataChangeEvent 的发送，而这个事件会被 AsyncNotifyService 监听到，这个 AsyncNotifyService 会根据节点的健康情况发送给其它的对等节点（B1,B2,B3），这个是通过内部接口 communication 来发送请求的  
2. 其它节点：得到http请求后，这个communication接口做了两件事，一是 dump：即根据是否是集群模式，如果是集群模式来将配置信息写入到硬盘（TODO：为什么硬盘？不写MYSQL吗？），二是 TaskMgr 0.1s 死轮询关键任务，这个关键任务就包括的事件改动的通知，这个任务内部做了 CacheItem 刷本节点的内存最新值，二是发布 LocalDataChangeEvent 事件给 LongPollingService 监听，其中 LongPollingService 内部会做 DataChangeTask 的任务来让响应 ClientLongPolling.sendResponse() 方法，从而让监听者即时返回  asyncContext.complete()  
修改会有点绕，因为里面的发布订阅和定时任务太多了
 
## 5.3 配置的删除
 
 1. 数据库物理删除 
 2. 发布 ConfigDataChangeEvent 事件
 
## 5.4 配置的获取
 
1. 直接获取本节点的 CacheItem 值，这个是 concurrenthashmap 类型，存储在内存中

## 5.5 配置的整体流程

![][3]  
https://github.com/LiWenGu/nacos.git  
https://www.processon.com/view/link/5d441f3fe4b0bc1bbedcf559

# 6. 代码地址

https://github.com/LiWenGu/nacos.git

[1]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/Nacos/536C41DF-52DF-4692-802E-AC3D537A434B.png
[2]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/Nacos/6015F579-FDC6-490D-9D0E-858D9BC2B827.png
[3]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/somephoto/Nacos%E9%85%8D%E7%BD%AE%E7%AE%A1%E7%90%86%E6%B5%81%E7%A8%8B.jpg