---
title: skywalking-service-traffic的交互
date: 2021-06-12 14:40:00
updated: 2021-06-12 14:40:00
tags:
  - skywalking
categories: 
  - [技术总结, apm]
comments: true
permalink: tech/apm/skywalking_2_service-traffic.html    
---

# 0. 背景

此文源于在二开 SkyWalking 时，发现公司搭建的 SkyWalking 当前服务列表有时候会显示不出来已经接入的应用，在排查和解决过程中的总结。
![和亲戚孩子拼的机器人][0]

<!--more-->

# 1. 后台界面显示位置

![后台服务列表][1]

在当前服务的列表栏，会显示 service_traffic 表的数据，在存储层的 service_traffic 数据格式如下:

![存储层服务流量表][2]

目前只发现有两个字段有用，一个是 name 字段，一个是 node_type 字段，而 node_type 在枚举 `org.apache.skywalking.oap.server.core.analysis.NodeType` 中定义：

```java
public enum NodeType {
    /**
     * <code>Normal = 0;</code>
     * This node type would be treated as an observed node.
     */
    Normal(0),
    /**
     * <code>Database = 1;</code>
     */
    Database(1),
    /**
     * <code>RPCFramework = 2;</code>
     */
    RPCFramework(2),
    /**
     * <code>Http = 3;</code>
     */
    Http(3),
    /**
     * <code>MQ = 4;</code>
     */
    MQ(4),
    /**
     * <code>Cache = 5;</code>
     */
    Cache(5),
    /**
     * <code>Browser = 6;</code>
     * This node type would be treated as an observed node.
     */
    Browser(6),
    /**
     * <code>User = 10</code>
     */
    User(10),
    /**
     * <code>Unrecognized = 11</code>
     */
    Unrecognized(11);
    ...
```

在发现有的应用接入 Skywalking 并启动后，但是在 `service_traffic` 表中没有显示，而相关的信息，例如 jvm 上报信息，以及实例的信息都是可以在数据中查到的。

>服务的实例相关信息可以在 `instance_traffic` 表中查询，其中 `service_id` 字段是 `service_traffic` 表的 `name` 字段的 base64 编码 + “.1”，例如 name 为 a1 的服务，它的 service_id 为：base64("a1") + ".1"。这里我也二开改过，原生会在后面加上随机数。

# 2. client 上传 service_traffic 信息

目前 client 会在上传 segment 的时候才会顺带将 service_traffic 信息上报，其实这里已经知道为什么我们应用接入 Skywalking，但是没有应用服务的信息，因为虽然产生了 jvm 以及实例机器的相关信息，但是没有产生让 plugin 拦截的 segment 上报导致。

以 kafka 为例，在客户端上传 segment 的地方，`org.apache.skywalking.apm.agent.core.kafka.KafkaTraceSegmentServiceClient` 该类负责客户端的 segment 的信息上传。topic 为 `skywalking-segment`。上报格式为 `apm-protocol/apm-network/src/main/proto/language-agent/Tracing.proto:50`：

```proto
message SegmentObject {
    string traceId = 1;

    string traceSegmentId = 2;

    repeated SpanObject spans = 3;

    // 上传的 appname
    string service = 4;

    string serviceInstance = 5;

    bool isSizeLimited = 6;
}
```

客户端批量一个一个的将 segment 上传后，等待 oap 的处理。

# 3. oap 接收 segment 信息，并顺带处理 service_traffic 信息

oap 服务端通过 `org.apache.skywalking.oap.server.analyzer.agent.kafka.provider.handler.TraceSegmentHandler` 接收来自客户端的 segment 信息。最终会走到 `ServiceTrafficDispatcher` 分发器类做分发处理：

```java
public class ServiceTrafficDispatcher implements SourceDispatcher<Service> {
    @Override
    public void dispatch(final Service source) {
        ServiceTraffic traffic = new ServiceTraffic();
        traffic.setTimeBucket(source.getTimeBucket());
        traffic.setName(source.getName());
        traffic.setNodeType(source.getNodeType());
        MetricsStreamProcessor.getInstance().in(traffic);
    }
}
```

这里涉及两个核心的类，一个是 `SourceDispatcher`，源数据处理分发器，它的实现类有很多，通过 `dispatch` 方法的入参 `org.apache.skywalking.oap.server.core.source.Service#scope` 方法来判断走哪个分发器，具体代码可以在 `org.apache.skywalking.oap.server.core.analysis.DispatcherManager` 中查看。

另一个核心的类是 `StreamProcessor` 流式处理器类，此类一般和 `SourceDispatcher` 结合使用，例如上面 `ServiceTrafficDispatcher` 就在获取到源数据后，通过 `MetricsStreamProcessor` 做流式处理。

>这里的流式处理较为复杂，涉及到 dataCarrier，多生产者多消费者，其中多生产者是各种数据通过流式处理塞入 dataCarrier，而多消费者指的是两个：AggregatorConsumer、PersistentConsumer，前者用于 metrics 指标聚合，后者用于将 metrics 指标存库。

在流式处理过程中，会走到 `org.apache.skywalking.oap.server.core.analysis.worker.MetricsRemoteWorker` 该 metrics 分析方法。该方法用于是否将当前的 metrics 信息传递给其它的 oap 角色。注意：`service_traffic` 也是 metrics 的一种。通过 `service_traffic` 表字段 `time_bucket` 也可以猜测的到，它也有自己的时间桶。

# 4. oap 存 service_traffic 入库过程

当 `service_traffic` 作为 metrics 的一种存入 datacarrier 后，会被 `org.apache.skywalking.oap.server.core.analysis.worker.MetricsPersistentWorker.PersistentConsumer` 取出并消费，将一段时间的客户端的 `service_triffic` 信息聚合，并存入 `org.apache.skywalking.oap.server.core.analysis.data.ReadWriteSafeCache` 该缓存中，而 `org.apache.skywalking.oap.server.core.storage.PersistenceTimer` 会通过定时任务，定时的取出 `ReadWriteSafeCache` 中的 metrics 信息，当然也包括了 `service_traffic` 信息。最终通过 `org.apache.skywalking.oap.server.core.analysis.worker.MetricsPersistentWorker#flushDataToStorage` 存入数据库，这里看下该方法源码：

```java
// 此方法用于所有 metrics 的存库方法，不仅仅是 service_traffic 信息。
private void  flushDataToStorage(List<Metrics> metricsList,
                                    List<PrepareRequest> prepareRequests) {
    try {
        // 先从数据库中获取该次 metrics 信息，并存入 context 中
        loadFromStorage(metricsList);
        for (Metrics metrics : metricsList) {
            // 判断当前 metrics 是否已经在数据库中有了
            Metrics cachedMetrics = context.get(metrics);
            if (cachedMetrics != null) {
                /*
                    * If the metrics is not supportUpdate, defined through MetricsExtension#supportUpdate,
                    * then no merge and further process happens.
                    */
                // service_traffic 就是不允许更新的，只允许插入，也就是说
                // 某个应用只有接入过 skywalking，并写入了 service_traffic
                // 那么永远都会有该应用的列表，即便后面卸载了 skywalking
                if (!supportUpdate) {
                    continue;
                }
                /*
                    * Merge metrics into cachedMetrics, change only happens inside cachedMetrics.
                    */
                // 将数据库中的 metrics 信息和当前实时获取的 metrics 信息做聚合
                cachedMetrics.combine(metrics);
                cachedMetrics.calculate();
                // instance_traffic 表更新客户端实例信息
                prepareRequests.add(metricsDAO.prepareBatchUpdate(model, cachedMetrics));
                nextWorker(cachedMetrics);
            } else {
                // service_traffic 写入es
                metrics.calculate();
                prepareRequests.add(metricsDAO.prepareBatchInsert(model, metrics));
                nextWorker(metrics);
            }

            /*
                * The `metrics` should be not changed in all above process. Exporter is an async process.
                */
            nextExportWorker.ifPresent(exportEvenWorker -> exportEvenWorker.in(
                new ExportEvent(metrics, ExportEvent.EventType.INCREMENT)));
        }
    } catch (Throwable t) {
        log.error(t.getMessage(), t);
    } finally {
        metricsList.clear();
    }
}
```

至此，oap 就将 `service_traffic`(metrics 的一种)，获取并存入了库。

# 5. 结语

从上面分析可看出，`service_traffic` 是不大合理的，有以下几点：

1. 只有插入，没有更新和删除
2. 只有在客户端 segment 上报时，该信息才会上报。

因此对该不合理的地方做了二开，一是利用 `service_traffic` 的 time_bucket 字段，实现更新和删除。二是在客户端注册的地方，触发 `service_traffic` 的流式处理，而不是在 segment 上报的时候。

[0]: https://markdownnoteimages.oss-cn-hangzhou.aliyuncs.com/90ECDCD7-2CE1-4FBD-B7F0-E9979FD42F0B_1_105_c.jpeg
[1]: https://markdownnoteimages.oss-cn-hangzhou.aliyuncs.com/20210612145841.png
[2]: https://markdownnoteimages.oss-cn-hangzhou.aliyuncs.com/20210612150120.png