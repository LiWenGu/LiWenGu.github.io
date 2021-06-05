---
title: skywalking-profile原理
date: 2021-05-29 15:52:00
updated: 2021-05-29 15:52:00
tags:
  - dubbo
categories: 
  - [技术总结, apm]
comments: true
permalink: tech/apm/skywalking_1_profile.html    
---

# 0. 背景

此文源于在二开 SkyWalking 时，发现公司搭建的 SkyWalking 性能剖析不能生效，在排查过程中学到了很多，因此记录下来方便对 SkyWalking 的性能剖析功能有更深的理解。

<!--more-->

# 1. 最开始的地方：后台创建性能剖析任务

![创建性能剖析任务][1]

## 1.1 后台接收 profile 任务创建请求

当创建任务时，oap 服务会通过内嵌的 org.apache.skywalking.oap.server.library.server.jetty.JettyJsonHandler.doPost 处理。而 GraphQLQueryHandler 是实际处理类，因为 GraphQLQueryHandler 继承自 JettyJsonHandler 类。

这是 org.apache.skywalking.oap.query.graphql.GraphQLQueryHandler#execute 源码：

```java
/**
     * 处理 oap 的 http post 请求
     * 以性能剖析任务创建为例，这是 graphql 的定义：server-query-plugin/query-graphql-plugin/resources/query-protocol/profile.graphqls
     * 而真正执行的 java 代码为：org.apache.skywalking.oap.query.graphql.resolver.ProfileMutation#createProfileTask()
     * @param request    oap 的后台查询语法有两个参数，一个是 query，包含要执行的类方法
     * @param variables  还有一个是 variables 参数，包含要执行的参数
     * @return 
     */
    private JsonObject execute(String request, Map<String, Object> variables) {
        try {
            ExecutionInput executionInput = ExecutionInput.newExecutionInput()
                                                          .query(request)
                                                          .variables(variables)
                                                          .build();
            // 通过 graphQL 语法找到真正执行的类方法，性能剖析的创建处理类为：ProfileMutation#createProfileTask()
            ExecutionResult executionResult = graphQL.execute(executionInput);
            LOGGER.debug("Execution result is {}", executionResult);
            Object data = executionResult.getData();
            List<GraphQLError> errors = executionResult.getErrors();
            JsonObject jsonObject = new JsonObject();
            if (data != null) {
                jsonObject.add(DATA, gson.fromJson(gson.toJson(data), JsonObject.class));
            }

            if (CollectionUtils.isNotEmpty(errors)) {
                JsonArray errorArray = new JsonArray();
                errors.forEach(error -> {
                    JsonObject errorJson = new JsonObject();
                    errorJson.addProperty(MESSAGE, error.getMessage());
                    errorArray.add(errorJson);
                });
                jsonObject.add(ERRORS, errorArray);
            }
            return jsonObject;
        } catch (final Throwable e) {
            ...
        }
    }
```

## 1.2 从 graphqls 定位到具体的执行类

我们看下 http 的入参：

```json
{
    "query": "mutation createProfileTask($creationRequest: ProfileTaskCreationRequest) {\n  createTask: createProfileTask(creationRequest: $creationRequest) {\n    id\n    errorReason\n  }\n  }",
    "variables": {
        "creationRequest": {
            "serviceId": "ZHViYm8tZXhhbXBsZS1jb25zdW1lcg==.1",
            "endpointName": "/api/users",
            "startTime": 1622276464546,
            "duration": 5,
            "minDurationThreshold": 0,
            "dumpPeriod": 10,
            "maxSamplingCount": 5
        }
    }
}
```

得到了要执行的类方法以及参数。
接着看

server-query-plugin/query-graphql-plugin/resources/query-protocol/profile.graphqls 内容：

```graphqls
...
extend type Mutation {
    # crate new profile task
    createProfileTask(creationRequest: ProfileTaskCreationRequest): ProfileTaskCreationResult!
}
...
```

最后定位到 ProfileMutation 类：

```java
/**
 * profile mutation GraphQL resolver
 */
public class ProfileMutation implements GraphQLMutationResolver {

    ...

    public ProfileTaskCreationResult createProfileTask(ProfileTaskCreationRequest creationRequest) throws IOException {
        return getProfileTaskService().createTask(creationRequest.getServiceId(), creationRequest.getEndpointName() == null ? null : creationRequest
            .getEndpointName()
            .trim(), creationRequest.getStartTime() == null ? -1 : creationRequest.getStartTime(), creationRequest.getDuration(), creationRequest
            .getMinDurationThreshold(), creationRequest.getDumpPeriod(), creationRequest.getMaxSamplingCount());
    }
}
```

## 1.3 profile task 任务存储

最终会将任务存储到 DB 中，以 ES7 存储为例：

存储的数据结构 ProfileTaskRecord：

```java
@ScopeDeclaration(id = PROFILE_TASK, name = "ProfileTask")
@Stream(name = ProfileTaskRecord.INDEX_NAME, scopeId = PROFILE_TASK, builder = ProfileTaskRecord.Builder.class, processor = NoneStreamProcessor.class)
public class ProfileTaskRecord extends NoneStream {

    public static final String INDEX_NAME = "profile_task";
    public static final String SERVICE_ID = "service_id";
    public static final String ENDPOINT_NAME = "endpoint_name";
    public static final String START_TIME = "start_time";
    public static final String DURATION = "duration";
    public static final String MIN_DURATION_THRESHOLD = "min_duration_threshold";
    public static final String DUMP_PERIOD = "dump_period";
    public static final String CREATE_TIME = "create_time";
    public static final String MAX_SAMPLING_COUNT = "max_sampling_count";
```

在 ES 的数据：

![ES中的profile数据][2]

# 2. client 将 profile 数据传输给 oap

当 oap 服务端创建了 profile task 后，需要让客户端感知，并将对应的 endpoint 的堆栈数据发送给 oap 服务端，以供分析。

## 2.1 client 请求 oap 创建的 profile task

agent client 通过 org.apache.skywalking.apm.agent.core.profile.ProfileTaskChannelService 中的定时任务来定时获取后端中的 profile task 列表，如下是 boot 方法的源码：

```java
@Override
public void boot() {
    sender = ServiceManager.INSTANCE.findService(ProfileSnapshotSender.class);
    // 客户端开启 profile 的开关
    if (Config.Profile.ACTIVE) {
        // query task list，定时从 oap 获取要分析的 endpoint 列表
        getTaskListFuture = Executors.newSingleThreadScheduledExecutor(
            new DefaultNamedThreadFactory("ProfileGetTaskService")
        ).scheduleWithFixedDelay(
            new RunnableWithExceptionProtection(
                this,
                t -> LOGGER.error("Query profile task list failure.", t)
            ), 0, Config.Collector.GET_PROFILE_TASK_INTERVAL, TimeUnit.SECONDS
        );
        // 将客户端收集的分析堆栈信息传输给 oap
        sendSnapshotFuture = Executors.newSingleThreadScheduledExecutor(
            new DefaultNamedThreadFactory("ProfileSendSnapshotService")
        ).scheduleWithFixedDelay(
            new RunnableWithExceptionProtection(
                () -> {
                    List<TracingThreadSnapshot> buffer = new ArrayList<>(Config.Profile.SNAPSHOT_TRANSPORT_BUFFER_SIZE);
                    snapshotQueue.drainTo(buffer);
                    if (!buffer.isEmpty()) {
                        sender.send(buffer);
                    }
                },
                t -> LOGGER.error("Profile segment snapshot upload failure.", t)
            ), 0, 500, TimeUnit.MILLISECONDS
        );
    }
}
```

从 oap 服务端获取 profile task 列表的执行逻辑 ProfileTaskChannelService.java：

```java
@Override
public void run() {
    if (status == GRPCChannelStatus.CONNECTED) {
        try {
            ProfileTaskCommandQuery.Builder builder = ProfileTaskCommandQuery.newBuilder();

            // sniffer info
            builder.setService(Config.Agent.SERVICE_NAME).setServiceInstance(Config.Agent.INSTANCE_NAME);

            // last command create time
            builder.setLastCommandTime(ServiceManager.INSTANCE.findService(ProfileTaskExecutionService.class)
                                                                .getLastCommandCreateTime());
            // 主动跟 oap 通信，通过 grpc 查询当前应用需要开启的 profile task 列表
            Commands commands = profileTaskBlockingStub.withDeadlineAfter(GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS)
                                                        .getProfileTaskCommands(builder.build());
            // 获取到任务列表后客户端处理该命令逻辑
            ServiceManager.INSTANCE.findService(CommandService.class).receiveCommand(commands);
        } catch (Throwable t) {
            if (!(t instanceof StatusRuntimeException)) {
                LOGGER.error(t, "Query profile task from backend fail.");
                return;
            }
            final StatusRuntimeException statusRuntimeException = (StatusRuntimeException) t;
            if (statusRuntimeException.getStatus().getCode() == Status.Code.UNIMPLEMENTED) {
                LOGGER.warn("Backend doesn't support profiling, profiling will be disabled");
                if (getTaskListFuture != null) {
                    getTaskListFuture.cancel(true);
                }

                // stop snapshot sender
                if (sendSnapshotFuture != null) {
                    sendSnapshotFuture.cancel(true);
                }
            }
        }
    }
}
```

## 2.2 oap 响应 client 获取 profile task 列表的请求

而 oap 是在 server-receiver-plugin/skywalking-profile-receiver-plugin 处理客户端的 ProfileTaskCommandQuery 请求，ProfileTaskServiceHandler.java：

```java
@Override
public void getProfileTaskCommands(ProfileTaskCommandQuery request, StreamObserver<Commands> responseObserver) {
    // query profile task list by service id
    final String serviceId = IDManager.ServiceID.buildId(request.getService(), NodeType.Normal);
    final String serviceInstanceId = IDManager.ServiceInstanceID.buildId(serviceId, request.getServiceInstance());
    // 从缓存中获取该服务的 profile task 列表，该缓存每隔 10s 从存储中更新 profile task
    final List<ProfileTask> profileTaskList = profileTaskCache.getProfileTaskList(serviceId);
    if (CollectionUtils.isEmpty(profileTaskList)) {
        responseObserver.onNext(Commands.newBuilder().build());
        responseObserver.onCompleted();
        return;
    }

    // build command list
    final Commands.Builder commandsBuilder = Commands.newBuilder();
    final long lastCommandTime = request.getLastCommandTime();

    for (ProfileTask profileTask : profileTaskList) {
        // if command create time less than last command time, means sniffer already have task
        if (profileTask.getCreateTime() <= lastCommandTime) {
            continue;
        }

        // record profile task log
        recordProfileTaskLog(profileTask, serviceInstanceId, ProfileTaskLogOperationType.NOTIFIED);

        // add command
        commandsBuilder.addCommands(commandService.newProfileTaskCommand(profileTask).serialize().build());
    }

    responseObserver.onNext(commandsBuilder.build());
    responseObserver.onCompleted();
}
```

## 2.3 client 开启 profile task 的相关定时任务

此时 client 主动向 oap 请求获取 profile task 列表，当 client 获取到后，需要进步做处理，ProfileTaskCommandExecutor.java：

```java
@Override
public void execute(BaseCommand command) throws CommandExecutionException {
    final ProfileTaskCommand profileTaskCommand = (ProfileTaskCommand) command;

    // build profile task
    final ProfileTask profileTask = new ProfileTask();
    profileTask.setTaskId(profileTaskCommand.getTaskId());
    profileTask.setFirstSpanOPName(profileTaskCommand.getEndpointName());
    profileTask.setDuration(profileTaskCommand.getDuration());
    profileTask.setMinDurationThreshold(profileTaskCommand.getMinDurationThreshold());
    profileTask.setThreadDumpPeriod(profileTaskCommand.getDumpPeriod());
    profileTask.setMaxSamplingCount(profileTaskCommand.getMaxSamplingCount());
    profileTask.setStartTime(profileTaskCommand.getStartTime());
    profileTask.setCreateTime(profileTaskCommand.getCreateTime());

    // send to executor
    // 将该任务列表塞入一个 list 中，该 list
    ServiceManager.INSTANCE.findService(ProfileTaskExecutionService.class).addProfileTask(profileTask);
}
```

当客户端获取到 profile task 后，需要知道什么时候启动，和什么时候停止，ProfileTaskExecutionService.java：

```java

/**
    * add profile task from OAP
    */
public void addProfileTask(ProfileTask task) {
    // update last command create time
    if (task.getCreateTime() > lastCommandCreateTime) {
        lastCommandCreateTime = task.getCreateTime();
    }

    // check profile task limit
    final CheckResult dataError = checkProfileTaskSuccess(task);
    if (!dataError.isSuccess()) {
        LOGGER.warn(
            "check command error, cannot process this profile task. reason: {}", dataError.getErrorReason());
        return;
    }

    // add task to list
    profileTaskList.add(task);

    // schedule to start task，计算出该任务还需要多久才能启动，通过定时任务实现
    long timeToProcessMills = task.getStartTime() - System.currentTimeMillis();
    PROFILE_TASK_SCHEDULE.schedule(() -> processProfileTask(task), timeToProcessMills, TimeUnit.MILLISECONDS);
}

/**
    * active the selected profile task to execution task, and start a removal task for it.
    */
private synchronized void processProfileTask(ProfileTask task) {
    // make sure prev profile task already stopped
    stopCurrentProfileTask(taskExecutionContext.get());

    // make stop task schedule and task context，通过上下文保存该 profile task，该上下文也实际在 trace 记录时塞入堆栈信息
    final ProfileTaskExecutionContext currentStartedTaskContext = new ProfileTaskExecutionContext(task);
    taskExecutionContext.set(currentStartedTaskContext);

    // start profiling this task，开启 profileing 定时任务，抓取堆栈信息
    currentStartedTaskContext.startProfiling(PROFILE_EXECUTOR);
    // 计算出该 profile task 的停止时间，通过定时任务停止该 profile task
    PROFILE_TASK_SCHEDULE.schedule(
        () -> stopCurrentProfileTask(currentStartedTaskContext), task.getDuration(), TimeUnit.MINUTES);
}
```

此时 client 已经开启了 profile task，并通过 oap 的参数，控制了开启时间和持续时间（停止时间）的定时任务，同时也开启了子线程 profile_executor，用于真正的抓取执行中的堆栈信息。也就是 client 开启了三个定时任务，一个用于 profile 的开启，一个用于 profile 的关闭，一个是用于 profile 真正抓取堆栈信息。

# 3 client 与 oap 的 profileSnapshot 信息

## 3.1 client profileSnapshot 的生成

在 ProfileTaskExecutionService.processProfileTask 方法中会开启 ProfileThread 任务，profileThread 定时任务会抓取堆栈信息并塞入队列。 ProfileThread.java：

```java
@Override
public void run() {

    try {
        profiling(taskExecutionContext);
    } catch (InterruptedException e) {
        // ignore interrupted
        // means current task has stopped
    } catch (Exception e) {
        LOGGER.error(e, "Profiling task fail. taskId:{}", taskExecutionContext.getTask().getTaskId());
    } finally {
        // finally stop current profiling task, tell execution service task has stop
        profileTaskExecutionService.stopCurrentProfileTask(taskExecutionContext);
    }

}

/**
    * start profiling
    */
private void profiling(ProfileTaskExecutionContext executionContext) throws InterruptedException {

    int maxSleepPeriod = executionContext.getTask().getThreadDumpPeriod();

    // run loop when current thread still running
    long currentLoopStartTime = -1;
    while (!Thread.currentThread().isInterrupted()) {
        currentLoopStartTime = System.currentTimeMillis();

        // each all slot
        AtomicReferenceArray<ThreadProfiler> profilers = executionContext.threadProfilerSlots();
        int profilerCount = profilers.length();
        for (int slot = 0; slot < profilerCount; slot++) {
            ThreadProfiler currentProfiler = profilers.get(slot);
            if (currentProfiler == null) {
                continue;
            }

            switch (currentProfiler.profilingStatus().get()) {

                case PENDING:
                    // check tracing context running time
                    currentProfiler.startProfilingIfNeed();
                    break;

                case PROFILING:
                    // dump stack
                    TracingThreadSnapshot snapshot = currentProfiler.buildSnapshot();
                    if (snapshot != null) {
                        // 将获取的堆栈信息快照文件，存入 snapshotQueue 队列中，而 ProfileTaskChannelService 中的 ProfileSendSnapshotService
                        // 定时任务会定期将 snapshotQueue 队列通过 ProfileSnapshotSender 任务发送给 oap
                        profileTaskChannelService.addProfilingSnapshot(snapshot);
                    } else {
                        // tell execution context current tracing thread dump failed, stop it
                        executionContext.stopTracingProfile(currentProfiler.tracingContext());
                    }
                    break;

            }
        }

        // sleep to next period
        // if out of period, sleep one period
        long needToSleep = (currentLoopStartTime + maxSleepPeriod) - System.currentTimeMillis();
        needToSleep = needToSleep > 0 ? needToSleep : maxSleepPeriod;
        Thread.sleep(needToSleep);
    }
}
```

将 profileSnapshot 信息由 client 发送给 oap，有两种方式，当前默认为 grpc 方式：ProfileSnapshotSender.java，也可以用 kafka 方式发送：KafkaProfileSnapshotSender.java

## 3.1 oap 获取 client 传输的 profileSnapshot 信息

grpc 方式获取也是通过 skywalking-profile-receiver-plugin 模块处理，该模块不仅负责 client 获取 profile task，也负责 profileSnapshot 的 grpc 通信，ProfileTaskServiceHandler.java：

```java
@Override
// 前文说过，oap 处理 client 获取 profile task 列表的请求处理
public void getProfileTaskCommands(ProfileTaskCommandQuery request, StreamObserver<Commands> responseObserver) {
}

 /**
    * oap 接收来自 client 的 profileSnapshot 信息
    * @param responseObserver
    * @return
    */
@Override
public StreamObserver<ThreadSnapshot> collectSnapshot(StreamObserver<Commands> responseObserver) {
    return new StreamObserver<ThreadSnapshot>() {
        @Override
        public void onNext(ThreadSnapshot snapshot) {
            if (LOGGER.isDebugEnabled()) {
                LOGGER.debug("receive profile segment snapshot");
            }

            // build database data
            final ProfileThreadSnapshotRecord record = new ProfileThreadSnapshotRecord();
            record.setTaskId(snapshot.getTaskId());
            record.setSegmentId(snapshot.getTraceSegmentId());
            record.setDumpTime(snapshot.getTime());
            record.setSequence(snapshot.getSequence());
            record.setStackBinary(snapshot.getStack().toByteArray());
            record.setTimeBucket(TimeBucket.getRecordTimeBucket(snapshot.getTime()));

            // async storage
            RecordStreamProcessor.getInstance().in(record);
        }

        @Override
        public void onError(Throwable throwable) {
            LOGGER.error(throwable.getMessage(), throwable);
            responseObserver.onCompleted();
        }

        @Override
        public void onCompleted() {
            responseObserver.onNext(Commands.newBuilder().build());
            responseObserver.onCompleted();
        }
    };
}
```

kafka 的 oap 接收profileSnapshot 也是类似处理，只不过换成了从 topic 中获取数据而已。

# 4. profileSnapshot 生成与分析

## 4.1 client 生成 profileSnapshot

在每次创建 TracingContext 的时候，也会初始化 profile 当前的堆栈信息：

```java
/**
    * Initialize all fields with default value.
    */
TracingContext(String firstOPName) {
    this.segment = new TraceSegment();
    this.spanIdGenerator = 0;
    isRunningInAsyncMode = false;
    createTime = System.currentTimeMillis();
    running = true;

    // profiling status
    if (PROFILE_TASK_EXECUTION_SERVICE == null) {
        PROFILE_TASK_EXECUTION_SERVICE = ServiceManager.INSTANCE.findService(ProfileTaskExecutionService.class);
    }
    // 判断是否需要 profile，同时尝试初始化当前 segment 的 thread 信息
    this.profileStatus = PROFILE_TASK_EXECUTION_SERVICE.addProfiling(
        this, segment.getTraceSegmentId(), firstOPName);

    this.correlationContext = new CorrelationContext();
    this.extensionContext = new ExtensionContext();
}
``

addProfiling 方法最终会走到 ProfileTaskExecutionContext.attemptProfiling 方法，该方法会将当前线程堆栈信息存储 volatile 变量：profilingSegmentSlots 中，该变量会在 ProfileThread.profiling 方法中通过 executionContext.threadProfilerSlots(); 方式死循环取出来。

```java
/**
    * check have available slot to profile and add it
    *
    * @return is add profile success
    */
public ProfileStatusReference attemptProfiling(TracingContext tracingContext,
                                                String traceSegmentId,
                                                String firstSpanOPName) {
    // check has available slot
    final int usingSlotCount = currentProfilingCount.get();
    if (usingSlotCount >= Config.Profile.MAX_PARALLEL) {
        return ProfileStatusReference.createWithNone();
    }

    // check first operation name matches
    if (!Objects.equals(task.getFirstSpanOPName(), firstSpanOPName)) {
        return ProfileStatusReference.createWithNone();
    }

    // if out limit started profiling count then stop add profiling
    if (totalStartedProfilingCount.get() > task.getMaxSamplingCount()) {
        return ProfileStatusReference.createWithNone();
    }

    // try to occupy slot
    if (!currentProfilingCount.compareAndSet(usingSlotCount, usingSlotCount + 1)) {
        return ProfileStatusReference.createWithNone();
    }
    // 将当前线程 Thread.currentThread() 赋值给 ThreadProfiler 的 profilingThread 参数
    // 同时将 threadProfiler 存入 profilingSegmentSlots 中，profilingSegmentSlots 会被 profileThread 取出
    final ThreadProfiler threadProfiler = new ThreadProfiler(
        tracingContext, traceSegmentId, Thread.currentThread(), this);
    int slotLength = profilingSegmentSlots.length();
    for (int slot = 0; slot < slotLength; slot++) {
        if (profilingSegmentSlots.compareAndSet(slot, null, threadProfiler)) {
            return threadProfiler.profilingStatus();
        }
    }
    return ProfileStatusReference.createWithNone();
}
```

例如当前只有一个 http 请求：

1. 在 before method 前将该请求堆栈信息初始化并发送给 oap，该信息包含该 traceId 以及 beginTime。
2. 该请求执行完后，在 after method 后，会将该 trace 发送给 oap。
3. oap 后台查询到 profile 的 trace 后，点击执行按钮，会从存储中获取该请求堆栈信息，并通过 traceId 反查到 trace 链路，并通过 beginTime 和 trace 的 time，得到该堆栈的执行时间。注意：后台点击执行时才会主动分析堆栈的执行和相关信息。

## 4.2 oap 执行分析堆栈信息

oap 请求执行堆栈分析参数：

```json
{
    "query": "query getProfileAnalyze($segmentId: String!, $timeRanges: [ProfileAnalyzeTimeRange!]!) {\\n  getProfileAnalyze: getProfileAnalyze(segmentId: $segmentId, timeRanges: $timeRanges) {\\n    tip\\n    trees {\\n      elements {\\n        id\\n        parentId\\n        codeSignature\\n        duration\\n        durationChildExcluded\\n        count\\n      }\\n    }\\n  }\\n  }",
    "variables": {
        "segmentId": "f26ef21ccee943d1806c12cf54948c00.57.16225422981970000",
        "timeRanges": [
            {
                "start": 1622542298197,
                "end": 1622542301399
            }
        ]
    }
}
```

定位到 org.apache.skywalking.oap.server.core.profile.analyze.ProfileAnalyzer#analyze() 执行的分析：从 es 取出从 client 获取的 ProfileSnapshot 堆栈信息，映射的实体类为：ProfileThreadSnapshotRecord

[1]: https://markdownnoteimages.oss-cn-hangzhou.aliyuncs.com/20210529161727.png
[2]: https://markdownnoteimages.oss-cn-hangzhou.aliyuncs.com/20210529171849.png