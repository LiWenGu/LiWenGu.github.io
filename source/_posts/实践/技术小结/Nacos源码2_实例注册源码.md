---
title: Nacos源码2_服务与实例注册源码
date: 2019-08-10 18:50:10
updated: 2018-06-18 18:50:10
tags:
  - 中间件
  - Nacos
categories: 
  - 小结
  - Nacos
comments: true
permalink: tech/nacos1_service_instance_1.html    
---

# 0. 启动调试 console

1. 加入 VM option：`-Dnacos.standalone=true
-Dnacos.functionMode=naming` 
2. 启动 console 项目，查看 banner  
![9a2c4dc62b9fc2e7e1acdbff1f77b2cd.png](evernotecid://7C43DEBC-F54E-48C8-9331-D22177889D6C/appyinxiangcom/11126461/ENResource/p158)

# 1. service 与 instance 的逻辑关系

每个服务可以有多个实例，每个实例又区分为两种实例，一种为临时实例，一种是非临时实例，它们的关系在 ServiceManager 中保存，具体的 service、instance、cluster 类都在 `com.alibaba.nacos.api.naming.pojo` 包下，对外的接口主要是 service、instance 的 CRUD 操作。其中有个 cluster 的逻辑结构，属于 service 和 instanc 之间，对应的是某个 service 有多少个 instance，就有多少个 cluster

# 2. service 的操作

nacos 本身支持两种协议，一种是 Raft 一种是 Distro  
而 service 创建操作使用了 Raft 协议 

## 2.1 请求的是 Leader 节点时

先在本节点发布 service change 事件，接着通过 http 发送到其它节点，同步该服务创建操作，如果这个请求超过 5s，就认为该操作失败。里面使用了代理类 consistencyDelegate，除了临时节点，其它类型的一致性操作统一走 Raft 协议，最终会到 RaftCore.signalPublish() 方法，遍历其它注册中心节点，POST 请求 /v1/ns/raft/datum/commit 接口，该接口主要做了如下判断：对比请求节点的 term 是否比自己大，如果请求节点的 term 大于等于自己节点的 term ，就执行该任务，将服务信息同步到本节点

## 2.2 请求的是非 Leader 节点时

如果当前节点是非 leader，直接得到 leader 节点的 url，并请求 /v1/ns/raft/datum/ 接口，将 service 创建事件给 leader 处理。注意，本节点的 term 是没有变的，这种情况下 leader 接收到请求后 term 会增加，并同步到其它节点，其它节点判断 leader 的 term 和自己等于或大于时就执行该操作

```java
public void signalPublish(String key, Record value) throws Exception {
        // 注释：步骤1 如果当前不是 leader，直接将该请求给 leader
        if (!isLeader()) {
            JSONObject params = new JSONObject();
            params.put("key", key);
            params.put("value", value);
            Map<String, String> parameters = new HashMap<>(1);
            parameters.put("key", key);

            raftProxy.proxyPostLarge(getLeader().ip, API_PUB, params.toJSONString(), parameters);
            return;
        }

        try {
            OPERATE_LOCK.lock();
            long start = System.currentTimeMillis();
            final Datum datum = new Datum();
            datum.key = key;
            datum.value = value;
            if (getDatum(key) == null) {
                datum.timestamp.set(1L);
            } else {
                datum.timestamp.set(getDatum(key).timestamp.incrementAndGet());
            }

            JSONObject json = new JSONObject();
            json.put("datum", datum);
            json.put("source", peers.local());
            // 注释：本节点发布 service 发布事件
            onPublish(datum, peers.local());

            final String content = JSON.toJSONString(json);

            final CountDownLatch latch = new CountDownLatch(peers.majorityCount());
            for (final String server : peers.allServersIncludeMyself()) {
                // 注释：这里应该是避免特殊情况：步骤1 之前正在选举 leader，在这里之间选举出了 leader
                if (isLeader(server)) {
                    latch.countDown();
                    continue;
                }
                final String url = buildURL(server, API_ON_PUB);
                HttpClient.asyncHttpPostLarge(url, Arrays.asList("key=" + key), content, new AsyncCompletionHandler<Integer>() {
                    @Override
                    public Integer onCompleted(Response response) throws Exception {
                        if (response.getStatusCode() != HttpURLConnection.HTTP_OK) {
                            Loggers.RAFT.warn("[RAFT] failed to publish data to peer, datumId={}, peer={}, http code={}",
                                datum.key, server, response.getStatusCode());
                            return 1;
                        }
                        latch.countDown();
                        return 0;
                    }

                    @Override
                    public STATE onContentWriteCompleted() {
                        return STATE.CONTINUE;
                    }
                });

            }   
            // 注释：超时 5s 就算失败，但是其实本节点已经执行了 service 创建事件
            if (!latch.await(UtilsAndCommons.RAFT_PUBLISH_TIMEOUT, TimeUnit.MILLISECONDS)) {
                // only majority servers return success can we consider this update success
                Loggers.RAFT.error("data publish failed, caused failed to notify majority, key={}", key);
                throw new IllegalStateException("data publish failed, caused failed to notify majority, key=" + key);
            }

            long end = System.currentTimeMillis();
            Loggers.RAFT.info("signalPublish cost {} ms, key: {}", (end - start), key);
        } finally {
            OPERATE_LOCK.unlock();
        }
    }
```

# 3. instance 的操作

instance 增加时，如果没有对应的 service，会默认创建该 service，如果已经有了 service ，会直接塞入到该 service 的 instanceList 中，具体细节在 ServiceManager 中进行操作。  
接着判断增加的 instance 是否临时来使用不同的一致性协议，如果为临时实例，使用 distro 协议，如果非临时实例，使用 raft 协议。distro 协议大致为定时任务广播其它节点+保存内存，其中广播的接口为 /distro/dump。
>distro 协议为自制协议，AP