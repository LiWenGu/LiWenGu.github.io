---
title: Nacos源码3_服务与实例注册源码
date: 2019-08-10 18:50:10
updated: 2018-06-18 18:50:10
tags:
  - 中间件
  - Nacos
categories: 
  - 实践
  - 技术小结
comments: true
permalink: tech/nacos1_service_instance_1.html    
---

心跳和选举主要依靠容器启动时的两个定时任务，分别为 HeartBeat 和 MasterElection 两个。这两个定时任务，每 0.5s 执行一次。  
当一段时间内（15~20s）没有接受到心跳，就会执行 MasterElection 里面的选举请求逻辑。  
```java
// 注释：master 选举
GlobalExecutor.registerMasterElection(new MasterElection());
// 注释：节点间心跳
GlobalExecutor.registerHeartbeat(new HeartBeat());

public class MasterElection implements Runnable {
    @Override
    public void run() {
        try {

            if (!peers.isReady()) {
                return;
            }

            RaftPeer local = peers.local();
            // 注释：每次递减0.5s，直到小于0.5s，初始是15~20s的范围，这个任务0.5s执行一次，说明选举在没心跳最坏的情况
            // 是15~20s进行一次选举，即发投票。但是心跳任务每次都会重置这个 leaderDueMs 为 15s
            local.leaderDueMs -= GlobalExecutor.TICK_PERIOD_MS;

            if (local.leaderDueMs > 0) {
                return;
            }

            // reset timeout
            // 注释：重置选举时间间隔
            local.resetLeaderDue();
            // 注释：重置心跳时间间隔为5s
            local.resetHeartbeatDue();

            sendVote();
        } catch (Exception e) {
            Loggers.RAFT.warn("[RAFT] error while master election {}", e);
        }

    }
}

public class HeartBeat implements Runnable {
    @Override
    public void run() {
        try {

            if (!peers.isReady()) {
                return;
            }

            RaftPeer local = peers.local();
            // 注释：5s以上发一次心跳（初始随机0~5s）
            local.heartbeatDueMs -= GlobalExecutor.TICK_PERIOD_MS;
            if (local.heartbeatDueMs > 0) {
                return;
            }
            // 注释：重置心跳间隔
            local.resetHeartbeatDue();
            // 注释：发送心跳，里面重置了 leaderDueMs 时间，即当心跳一直正常时，不会发起master选举
            sendBeat();
        } catch (Exception e) {
            Loggers.RAFT.warn("[RAFT] error while sending beat {}", e);
        }

    }
}
```

参考了更加细节的文章：  
文章一：心跳的源码细节：https://www.jianshu.com/p/b0cdaa64688e    
文章二：选举的源码细节：https://www.jianshu.com/p/5a2d965174ae  

代码地址：https://github.com/LiWenGu/nacos.git