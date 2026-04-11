# Flink

> Apache Flink 官网：https://flink.apache.org/
> 官方文档：https://nightlies.apache.org/flink/flink-docs-stable/
>
> 本文基于 Flink 1.17 版本

![Flink Logo](https://flink.apache.org/img/logo/png/500/flink_squirrel_500.png)

---

## 第1章 Flink 概述

### 1.1 🔥 Flink 是什么

Apache Flink 是一个**框架**和**分布式处理引擎**，用于在**无界**和**有界**数据流上进行**有状态的计算**（Stateful Computations over Data Streams）。

![流处理架构](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/flink-application-sources-sinks.png)

**有界流 vs 无界流：**

| 特性 | 无界流（Unbounded） | 有界流（Bounded） |
|------|---------------------|-------------------|
| 定义 | 有开始，无结束 | 有开始，有结束 |
| 处理方式 | 持续处理，数据摄取后立即处理 | 可等所有数据到达后再处理 |
| 排序要求 | 需要有序摄取 | 所有数据可排序 |
| 典型场景 | 流处理 | 批处理 |

**有状态的流处理：**

把流处理需要的**额外数据保存成一个"状态"**，然后针对这条数据进行处理，并**更新状态**。

![有状态的流处理](https://flink.apache.org/img/function-state.png)

### 1.2 ⭐ Flink 特点

- 🔥 **高吞吐和低延迟**：每秒处理数百万事件，毫秒级延迟
- 🔥 **结果的准确性**：提供事件时间（Event-Time）和处理时间（Processing-Time）语义
- 🔥 **精确一次**（Exactly-Once）的状态一致性保证
- **可连接到最常用的外部系统**：Kafka、Hive、JDBC、HDFS、Redis 等
- **高可用**：与 K8S、YARN 紧密集成，支持快速故障恢复

### 1.3 🔥 Flink vs Spark Streaming

| 对比维度 | **Flink** | **Spark Streaming** |
|:--------:|-----------|---------------------|
| **计算模型** | 🔥 真正的流计算 | 微批处理（Mini-Batch） |
| **数据模型** | DataStream（事件序列） | DStream（小批RDD集合） |
| **时间语义** | 事件时间 + 处理时间 | 仅处理时间 |
| **窗口** | 多种灵活窗口 | 窗口必须是批次的整数倍 |
| **状态管理** | ✅ 内置状态管理 | ❌ 无内置状态 |
| **流式SQL** | ✅ 完整支持 | ❌ 不支持 |
| **延迟** | 毫秒级 | 秒级（批次间隔） |
| **吞吐量** | 高 | 高 |
| **容错** | Checkpoint（Chandy-Lamport） | Checkpoint（RDD Lineage） |

![有界流和无界流](https://nightlies.apache.org/flink/flink-docs-master/fig/bounded-unbounded.png)

> 🔥 **核心区别**：Flink 以**流处理为根本**，一个事件在一个节点处理完后可以直接发往下一个节点；Spark 是**批处理为根本**，将 DAG 划分为不同 Stage，一个完成后才计算下一个。

### 1.4 📝 Flink 分层 API

![Flink 分层API](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/levels_of_abstraction.svg)

从下到上四层：

1. **有状态流处理**（Stateful Stream Processing）：最底层 ProcessFunction API
2. **DataStream API**：核心API，提供 map、flatMap、window、join 等操作。Flink 1.12+ 已实现**流批一体**，DataSet API 已过时
3. **Table API**：以表为中心的声明式编程，类似关系模型
4. **SQL**：最高层抽象，以SQL查询表达式表现程序

---

## 第2章 Flink 快速上手

### 2.1 ⭐ 环境搭建

**Maven 依赖：**

```xml
<properties>
    <flink.version>1.17.0</flink.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-streaming-java</artifactId>
        <version>${flink.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-clients</artifactId>
        <version>${flink.version}</version>
    </dependency>
</dependencies>
```

### 2.2 🔥 WordCount 代码示例

**批处理（DataSet API - 已过时）：**

```java
public class BatchWordCount {
    public static void main(String[] args) throws Exception {
        // 1. 创建执行环境
        ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
        // 2. 读取文件
        DataSource<String> lineDS = env.readTextFile("input/words.txt");
        // 3. 转换计算
        FlatMapOperator<String, Tuple2<String, Long>> wordAndOne = lineDS
            .flatMap((String line, Collector<Tuple2<String, Long>> out) -> {
                String[] words = line.split(" ");
                for (String word : words) {
                    out.collect(Tuple2.of(word, 1L));
                }
            }).returns(Types.TUPLE(Types.STRING, Types.LONG));
        // 4. 按word分组，聚合
        AggregateOperator<Tuple2<String, Long>> sum = wordAndOne.groupBy(0).sum(1);
        // 5. 输出
        sum.print();
    }
}
```

**🔥 流处理（DataStream API - 推荐）：**

```java
public class StreamWordCount {
    public static void main(String[] args) throws Exception {
        // 1. 创建流式执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        // 2. 读取socket文本流
        DataStreamSource<String> lineStream = env.socketTextStream("hadoop102", 7777);
        // 3. 转换计算
        SingleOutputStreamOperator<Tuple2<String, Long>> wordAndOne = lineStream
            .flatMap((String line, Collector<Tuple2<String, Long>> out) -> {
                String[] words = line.split(" ");
                for (String word : words) {
                    out.collect(Tuple2.of(word, 1L));
                }
            }).returns(Types.TUPLE(Types.STRING, Types.LONG));
        // 4. 分组聚合
        KeyedStream<Tuple2<String, Long>, String> keyedStream = wordAndOne.keyBy(t -> t.f0);
        SingleOutputStreamOperator<Tuple2<String, Long>> sum = keyedStream.sum(1);
        // 5. 输出
        sum.print();
        // 6. 执行
        env.execute("Stream WordCount");
    }
}
```

---

## 第3章 Flink 部署

### 3.1 🔥 集群角色

![Flink 集群剖析](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/processes.svg)

| 角色 | 说明 |
|------|------|
| **客户端（Client）** | 提交作业，不参与运行时执行 |
| **JobManager** | 🔥 作业管理者，负责调度、Checkpoint 协调、故障恢复 |
| **TaskManager** | 🔥 任务执行者，负责数据处理，拥有 Task Slot |

### 3.2 🔥 部署模式

| 部署模式 | 集群生命周期 | main()运行位置 | 适用场景 |
|----------|-------------|---------------|----------|
| **会话模式（Session）** | 预先启动，共享 | 客户端 | 开发测试、小作业 |
| **单作业模式（Per-Job）** | 每个作业独立集群 | 客户端 | Flink 1.15 前生产环境 |
| **🔥 应用模式（Application）** | 每个作业独立集群 | JobManager | 🔥 **生产推荐**，减少网络传输 |

### 3.3 🔥 YARN 运行模式（重点）

**应用模式提交（生产推荐）：**

```bash
# 应用模式提交
bin/flink run-application -t yarn-application \
    -Djobmanager.memory.process.size=2048m \
    -Dtaskmanager.memory.process.size=4096m \
    -Dtaskmanager.numberOfTaskSlots=2 \
    -c com.atguigu.wc.StreamWordCount \
    FlinkTutorial-1.0.jar

# 查看运行中的作业
bin/flink list -t yarn-application -Dyarn.application.id=application_xxxx

# 取消作业
bin/flink cancel -t yarn-application -Dyarn.application.id=application_xxxx <jobId>
```

**会话模式提交：**

```bash
# 启动 YARN Session
bin/yarn-session.sh -nm test -d

# 提交作业
bin/flink run -c com.atguigu.wc.StreamWordCount FlinkTutorial-1.0.jar
```

---

## 第4章 Flink 运行时架构

### 4.1 🔥 核心概念

#### 并行度（Parallelism）

![并行度](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/parallel_dataflow.svg)

🔥 **并行度优先级**（从高到低）：
1. **算子级别**：`operator.setParallelism(2)`
2. **全局执行环境**：`env.setParallelism(2)`
3. **提交参数**：`-p 2`
4. **配置文件**：`flink-conf.yaml` 中 `parallelism.default: 1`

#### 算子链（Operator Chain）

![合并算子链](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/tasks_chains.svg)

🔥 **合并条件**：
- 上下游算子并行度相同
- 数据传输方式为 **forward**（一对一）
- 属于同一个 **SlotSharingGroup**

#### 🔥 任务槽（Task Slots）

- 每个 TaskManager 是一个 **JVM 进程**
- Task Slot 是 TaskManager 资源的**固定子集**（仅隔离内存，不隔离 CPU）
- `taskmanager.numberOfTaskSlots` 建议设置为 **CPU 核心数**
- 🔥 **Slot 共享**：同一作业的不同算子的子任务可以共享同一个 Slot

#### 🔥 四张图的演化

| 图 | 说明 |
|----|------|
| **StreamGraph** | 逻辑流图，Client 端根据用户代码生成 |
| **JobGraph** | 作业图，Client 端优化（算子链合并） |
| **ExecutionGraph** | 执行图，JobManager 根据并行度展开 |
| **Physical Graph** | 物理执行图，TaskManager 实际运行 |

---

## 第5章 DataStream API

### 5.1 ⭐ 执行环境

```java
// 自动识别运行环境（本地/集群）
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 执行模式设置（流批一体）
env.setRuntimeMode(RuntimeExecutionMode.STREAMING); // 默认流模式
env.setRuntimeMode(RuntimeExecutionMode.BATCH);      // 批模式
env.setRuntimeMode(RuntimeExecutionMode.AUTOMATIC);   // 自动识别
```

### 5.2 🔥 源算子（Source）

#### 从 Kafka 读取数据（最常用）

```java
KafkaSource<String> kafkaSource = KafkaSource.<String>builder()
    .setBootstrapServers("hadoop102:9092,hadoop103:9092,hadoop104:9092")
    .setTopics("topic_a")
    .setGroupId("atguigu")
    .setStartingOffsets(OffsetsInitializer.latest())
    .setValueOnlyDeserializer(new SimpleStringSchema())
    .build();

DataStreamSource<String> kafkaDS = env.fromSource(
    kafkaSource,
    WatermarkStrategy.noWatermarks(),
    "Kafka Source"
);
```

#### 从文件读取数据

```java
FileSource<String> fileSource = FileSource
    .forRecordStreamFormat(new TextLineInputFormat(), new Path("input/"))
    .build();

DataStreamSource<String> fileDS = env.fromSource(
    fileSource,
    WatermarkStrategy.noWatermarks(),
    "File Source"
);
```

#### 从数据生成器读取

```java
DataGeneratorSource<String> generatorSource = new DataGeneratorSource<>(
    new GeneratorFunction<Long, String>() {
        @Override
        public String map(Long value) throws Exception {
            return "Number:" + value;
        }
    },
    Long.MAX_VALUE,
    RateLimiterStrategy.perSecond(10),
    Types.STRING
);
```

### 5.3 🔥 转换算子（Transformation）

#### 基本转换算子

```java
// map：一对一转换
stream.map(value -> value * 2);

// filter：过滤
stream.filter(value -> value > 0);

// flatMap：一对多转换
stream.flatMap((String line, Collector<String> out) -> {
    for (String word : line.split(" ")) {
        out.collect(word);
    }
}).returns(Types.STRING);
```

#### 🔥 聚合算子

```java
// keyBy：按键分区（逻辑分区，不改变并行度）
KeyedStream<Event, String> keyedStream = stream.keyBy(e -> e.user);

// 简单聚合
keyedStream.sum("amount");
keyedStream.min("timestamp");  // 仅更新指定字段
keyedStream.minBy("timestamp"); // 更新整条数据

// reduce：归约聚合
keyedStream.reduce((value1, value2) -> {
    return new Event(value1.user, value2.url,
        Math.max(value1.timestamp, value2.timestamp));
});
```

![keyBy](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/keyBy.png)

#### 🔥 物理分区算子

```java
stream.shuffle();        // 随机分区
stream.rebalance();      // 轮询分区（解决数据倾斜）
stream.rescale();        // 本地轮询（效率更高）
stream.broadcast();      // 广播到所有分区
stream.global();         // 全局分区（发往第一个分区）
stream.partitionCustom(partitioner, keySelector); // 自定义分区
```

#### ⭐ 分流与合流

```java
// 使用侧输出流分流
OutputTag<Event> lateTag = new OutputTag<Event>("late"){};

SingleOutputStreamOperator<Event> mainStream = stream
    .process(new ProcessFunction<Event, Event>() {
        @Override
        public void processElement(Event value, Context ctx, Collector<Event> out) {
            if (value.timestamp > threshold) {
                out.collect(value);  // 主流
            } else {
                ctx.output(lateTag, value);  // 侧输出流
            }
        }
    });

DataStream<Event> lateStream = mainStream.getSideOutput(lateTag);

// Union：合并同类型流
stream1.union(stream2, stream3);

// Connect：连接不同类型流
ConnectedStreams<Integer, String> connected = intStream.connect(strStream);
```

![connect](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/connected-streams.svg)

### 5.4 🔥 输出算子（Sink）

#### 输出到 Kafka

```java
KafkaSink<String> kafkaSink = KafkaSink.<String>builder()
    .setBootstrapServers("hadoop102:9092")
    .setRecordSerializer(
        KafkaRecordSerializationSchema.builder()
            .setTopic("topic_b")
            .setValueSerializationSchema(new SimpleStringSchema())
            .build()
    )
    .setDeliveryGuarantee(DeliveryGuarantee.EXACTLY_ONCE) // 精确一次
    .setTransactionalIdPrefix("atguigu-")
    .build();

stream.sinkTo(kafkaSink);
```

#### 输出到 MySQL（JDBC）

```java
stream.addSink(JdbcSink.sink(
    "INSERT INTO clicks (user, url) VALUES (?, ?)",
    (statement, event) -> {
        statement.setString(1, event.user);
        statement.setString(2, event.url);
    },
    JdbcExecutionOptions.builder()
        .withBatchSize(1000)
        .withBatchIntervalMs(200)
        .withMaxRetries(5)
        .build(),
    new JdbcConnectionOptions.JdbcConnectionOptionsBuilder()
        .withUrl("jdbc:mysql://hadoop102:3306/test")
        .withDriverName("com.mysql.cj.jdbc.Driver")
        .withUsername("root")
        .withPassword("000000")
        .build()
));
```

#### 输出到文件

```java
FileSink<String> fileSink = FileSink
    .forRowFormat(new Path("output/"), new SimpleStringEncoder<String>("UTF-8"))
    .withRollingPolicy(
        DefaultRollingPolicy.builder()
            .withRolloverInterval(Duration.ofMinutes(15))
            .withInactivityInterval(Duration.ofMinutes(5))
            .withMaxPartSize(MemorySize.ofMebiBytes(1024))
            .build()
    )
    .build();

stream.sinkTo(fileSink);
```

---

## 第6章 时间与窗口

### 6.1 🔥 窗口（Window）

#### 窗口分类

![窗口概览](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/windows.svg)

| 窗口类型 | 说明 | 示意图 |
|----------|------|--------|
| **🔥 滚动窗口** | 固定大小，无重叠 | ![tumbling](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/tumbling-windows.svg) |
| **滑动窗口** | 固定大小，有重叠 | ![sliding](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/sliding-windows.svg) |
| **会话窗口** | 非活动间隔分隔 | ![session](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/session-windows.svg) |
| **全局窗口** | 所有数据在一个窗口 | ![global](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/non-windowed.svg) |

#### 🔥 窗口 API

```java
// 按键分区窗口
stream.keyBy(...)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))  // 滚动窗口
    .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))  // 滑动窗口
    .window(EventTimeSessionWindows.withGap(Time.seconds(10)))  // 会话窗口

// 非按键分区窗口
stream.windowAll(TumblingProcessingTimeWindows.of(Time.seconds(5)))

// 计数窗口
stream.keyBy(...)
    .countWindow(10)       // 滚动计数窗口
    .countWindow(10, 2)    // 滑动计数窗口
```

#### 🔥 窗口函数

```java
// 1. 增量聚合：ReduceFunction
.reduce((v1, v2) -> new Event(v1.user, v1.url, v2.timestamp));

// 2. 增量聚合：AggregateFunction（更灵活）
.aggregate(new AggregateFunction<Event, Long, Double>() {
    public Long createAccumulator() { return 0L; }
    public Long add(Event value, Long acc) { return acc + 1; }
    public Double getResult(Long acc) { return (double) acc; }
    public Long merge(Long a, Long b) { return a + b; }
});

// 3. 全窗口函数：ProcessWindowFunction（可获取窗口信息）
.process(new ProcessWindowFunction<Event, String, String, TimeWindow>() {
    public void process(String key, Context ctx, Iterable<Event> elements, Collector<String> out) {
        long count = 0;
        for (Event e : elements) count++;
        long start = ctx.window().getStart();
        long end = ctx.window().getEnd();
        out.collect("窗口 [" + start + ", " + end + ") key=" + key + " count=" + count);
    }
});

// 🔥 4. 增量 + 全窗口结合（最佳实践）
.aggregate(myAggFunction, myProcessWindowFunction);
```

### 6.2 🔥 时间语义

![时间语义](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/event_processing_time.svg)

| 时间语义 | 说明 | 适用场景 |
|----------|------|----------|
| **🔥 事件时间（Event Time）** | 数据自带的时间戳 | 生产环境推荐，结果确定 |
| **处理时间（Processing Time）** | 处理数据的机器时间 | 延迟最低，但结果不确定 |

### 6.3 🔥 水位线（Watermark）

**水位线本质：**
- 水位线是一个**时间戳**，表示"不会再有时间戳 ≤ 该水位线的数据到来"
- 用于触发窗口计算：当水位线 ≥ 窗口结束时间，窗口触发计算

![有序水位线](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/stream_watermark_in_order.svg)

![乱序水位线](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/stream_watermark_out_of_order.svg)

#### 🔥 水位线生成策略

```java
// 1. 有序流（无延迟）
WatermarkStrategy.<Event>forMonotonousTimestamps()
    .withTimestampAssigner((event, ts) -> event.timestamp);

// 2. 🔥 乱序流（设置最大延迟）—— 最常用
WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(5))
    .withTimestampAssigner((event, ts) -> event.timestamp);

// 3. 自定义水位线生成器
WatermarkStrategy.<Event>forGenerator(ctx -> new MyWatermarkGenerator())
    .withTimestampAssigner((event, ts) -> event.timestamp);
```

> 🔥 **水位线公式**：`Watermark = 当前最大事件时间 - 最大乱序程度 - 1ms`

#### 🔥 迟到数据三重保障

```java
// 第一重：水位线延迟（forBoundedOutOfOrderness）
WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(5))

// 第二重：窗口允许迟到
.window(TumblingEventTimeWindows.of(Time.seconds(10)))
.allowedLateness(Time.minutes(1))  // 窗口延迟关闭1分钟

// 第三重：侧输出流兜底
OutputTag<Event> lateTag = new OutputTag<Event>("late"){};
.sideOutputLateData(lateTag)
```

### 6.4 ⭐ 双流联结（Join）

#### 窗口联结（Window Join）

```java
stream1.join(stream2)
    .where(e1 -> e1.key)
    .equalTo(e2 -> e2.key)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .apply((e1, e2) -> e1 + " -> " + e2);
```

#### 间隔联结（Interval Join）

```java
// 订单流 join 支付流，支付在下单后 5秒~30分钟内
orderStream.keyBy(o -> o.orderId)
    .intervalJoin(payStream.keyBy(p -> p.orderId))
    .between(Time.seconds(5), Time.minutes(30))
    .process(new ProcessJoinFunction<Order, Pay, String>() {
        public void processElement(Order left, Pay right, Context ctx, Collector<String> out) {
            out.collect(left.orderId + " 已支付：" + right.amount);
        }
    });
```

---

## 第7章 处理函数（Process Function）

### 7.1 🔥 ProcessFunction

最底层 API，可以访问：**时间戳**、**水位线**、**定时器**、**侧输出流**、**状态**。

```java
stream.keyBy(e -> e.user)
    .process(new KeyedProcessFunction<String, Event, String>() {
        // 每条数据触发
        @Override
        public void processElement(Event value, Context ctx, Collector<String> out) {
            // 获取当前处理时间
            long currentTime = ctx.timerService().currentProcessingTime();
            // 注册10秒后的定时器
            ctx.timerService().registerProcessingTimeTimer(currentTime + 10000L);
            out.collect(value.toString());
        }

        // 定时器触发
        @Override
        public void onTimer(long timestamp, OnTimerContext ctx, Collector<String> out) {
            out.collect("定时器触发：" + timestamp + " key=" + ctx.getCurrentKey());
        }
    });
```

### 7.2 ⭐ Top N 案例

```java
// 使用 KeyedProcessFunction 实现实时 Top N
stream.keyBy(e -> e.url)
    .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .aggregate(new UrlCountAgg(), new UrlCountResult())  // 窗口聚合
    .keyBy(r -> r.windowEnd)  // 按窗口分组
    .process(new TopN(3));     // 取 Top 3
```

---

## 第8章 状态管理

### 8.1 🔥 状态分类

| 分类 | 说明 | 使用场景 |
|------|------|----------|
| **🔥 Keyed State** | 按 Key 隔离的状态，只能在 KeyedStream 上使用 | 大部分业务场景 |
| **Operator State** | 绑定到算子实例，与 Key 无关 | Kafka Consumer Offset |

### 8.2 🔥 按键分区状态（Keyed State）

| 状态类型 | 说明 | 典型场景 |
|----------|------|----------|
| **🔥 ValueState** | 存储单个值 | 上次登录时间、累计金额 |
| **🔥 ListState** | 存储列表 | 收集窗口数据 |
| **🔥 MapState** | 存储键值对 | 去重、计数 |
| **ReducingState** | 存储聚合结果 | 持续归约 |
| **AggregatingState** | 存储聚合结果（可变类型） | 复杂聚合 |

```java
public class MyKeyedProcessFunction extends KeyedProcessFunction<String, Event, String> {

    // 声明状态
    private ValueState<Long> countState;
    private MapState<String, Long> urlCountState;

    @Override
    public void open(Configuration parameters) {
        // 状态描述器
        countState = getRuntimeContext().getState(
            new ValueStateDescriptor<>("count", Long.class));
        urlCountState = getRuntimeContext().getMapState(
            new MapStateDescriptor<>("url-count", String.class, Long.class));
    }

    @Override
    public void processElement(Event value, Context ctx, Collector<String> out) throws Exception {
        // 使用 ValueState
        Long count = countState.value();
        countState.update(count == null ? 1L : count + 1);

        // 使用 MapState
        if (urlCountState.contains(value.url)) {
            urlCountState.put(value.url, urlCountState.get(value.url) + 1);
        } else {
            urlCountState.put(value.url, 1L);
        }
    }
}
```

#### ⭐ 状态生存时间（TTL）

```java
StateTtlConfig ttlConfig = StateTtlConfig.newBuilder(Time.hours(1))
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)  // 创建和写入时更新
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)  // 不返回过期值
    .build();

ValueStateDescriptor<Long> descriptor = new ValueStateDescriptor<>("count", Long.class);
descriptor.enableTimeToLive(ttlConfig);
```

### 8.3 🔥 状态后端（State Backends）

| 状态后端 | 状态存储位置 | 特点 | 适用场景 |
|----------|-------------|------|----------|
| **HashMapStateBackend** | JVM 堆内存 | 🔥 读写快，受内存限制 | 状态小、追求性能 |
| **🔥 EmbeddedRocksDBStateBackend** | RocksDB（本地磁盘） | 可存储超大状态 | **生产推荐** |

```java
// 代码中设置
env.setStateBackend(new EmbeddedRocksDBStateBackend());

// flink-conf.yaml 中设置
// state.backend: rocksdb
// state.checkpoints.dir: hdfs://hadoop102:8020/flink/checkpoints
```

---

## 第9章 容错机制

### 9.1 🔥 检查点（Checkpoint）

#### 核心原理

Flink 使用 **Chandy-Lamport 分布式快照算法**，通过在数据流中插入 **Barrier（分界线）** 来实现一致性快照。

| 概念 | 说明 |
|------|------|
| **Barrier** | 由 JobManager 注入到数据流中的特殊标记 |
| **Barrier 对齐** | 🔥 等待所有输入通道的 Barrier 到齐后才做快照 → **Exactly-Once** |
| **非 Barrier 对齐** | 不等待，直接做快照 → 减少延迟但增加状态大小 |
| **增量 Checkpoint** | 仅保存与上次的差异（RocksDB 支持） |

#### 🔥 Checkpoint 配置

```java
// 启用 Checkpoint，间隔 5 秒
env.enableCheckpointing(5000L);

// 设置精确一次语义
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

// Checkpoint 超时时间
env.getCheckpointConfig().setCheckpointTimeout(60000L);

// 两次 Checkpoint 之间最小间隔
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(2000L);

// 同时允许的最大 Checkpoint 数
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);

// 取消作业时保留 Checkpoint
env.getCheckpointConfig().setExternalizedCheckpointCleanup(
    CheckpointConfig.ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);

// 设置 Checkpoint 存储路径
env.getCheckpointConfig().setCheckpointStorage("hdfs://hadoop102:8020/flink/checkpoints");

// 开启非对齐 Checkpoint（减少反压时的延迟）
env.getCheckpointConfig().enableUnalignedCheckpoints();
```

#### ⭐ 保存点（Savepoint）

```bash
# 触发 Savepoint
bin/flink savepoint <jobId> hdfs://hadoop102:8020/flink/savepoints

# 从 Savepoint 恢复
bin/flink run -s hdfs://hadoop102:8020/flink/savepoints/savepoint-xxx \
    -c com.atguigu.MyJob myJob.jar

# 停止作业并触发 Savepoint
bin/flink stop --savepointPath hdfs://hadoop102:8020/flink/savepoints <jobId>
```

### 9.2 🔥 端到端精确一次（End-To-End Exactly-Once）

| 环节 | 保证方式 |
|------|----------|
| **输入端** | 🔥 Kafka 可重放（记录 Offset） |
| **Flink 内部** | 🔥 Checkpoint（Barrier 对齐） |
| **输出端** | 🔥 **两阶段提交（2PC）** / 幂等写入 |

**Flink + Kafka 端到端精确一次：**
1. Flink 开启 Checkpoint
2. Kafka Producer 设置 `EXACTLY_ONCE` 语义
3. Kafka Consumer 设置 `read_committed` 隔离级别
4. 事务超时时间：`transaction.max.timeout.ms` > Checkpoint 间隔

---

## 第10章 Flink SQL

### 10.1 ⭐ SQL-Client

```bash
# 基于 YARN Session 启动
bin/sql-client.sh -s yarn-session

# 常用配置
SET 'execution.runtime-mode' = 'streaming';
SET 'sql-client.execution.result-mode' = 'tableau';
SET 'parallelism.default' = '2';
```

### 10.2 🔥 动态表与持续查询

在流处理中，表是**动态表**（Dynamic Table）：
- 输入流 → **追加模式**转换为动态表
- 在动态表上执行 **持续查询**（Continuous Query）
- 查询结果也是动态表
- 动态表通过 **changelog 流**转换回 DataStream

| 模式 | 说明 |
|------|------|
| **Append-only** | 仅追加，INSERT 操作 |
| **Retract** | 撤回模式，INSERT + DELETE |
| **Upsert** | 更新插入，INSERT + UPDATE（需要主键） |

### 10.3 🔥 DDL 定义

#### Kafka 表

```sql
CREATE TABLE kafka_source (
    `user_id` STRING,
    `url` STRING,
    `ts` TIMESTAMP(3),
    WATERMARK FOR ts AS ts - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'clicks',
    'properties.bootstrap.servers' = 'hadoop102:9092',
    'properties.group.id' = 'atguigu',
    'scan.startup.mode' = 'latest-offset',
    'format' = 'json'
);
```

#### MySQL 表（JDBC）

```sql
CREATE TABLE mysql_sink (
    `user_id` STRING,
    `url_count` BIGINT,
    PRIMARY KEY (`user_id`) NOT ENFORCED
) WITH (
    'connector' = 'jdbc',
    'url' = 'jdbc:mysql://hadoop102:3306/test',
    'table-name' = 'url_count',
    'username' = 'root',
    'password' = '000000'
);
```

### 10.4 🔥 窗口 TVF 聚合

```sql
-- 🔥 滚动窗口 TVF（推荐）
SELECT
    window_start, window_end, user_id,
    COUNT(url) AS cnt
FROM TABLE(
    TUMBLE(TABLE kafka_source, DESCRIPTOR(ts), INTERVAL '10' SECOND)
)
GROUP BY window_start, window_end, user_id;

-- 滑动窗口 TVF
SELECT
    window_start, window_end, user_id,
    COUNT(url) AS cnt
FROM TABLE(
    HOP(TABLE kafka_source, DESCRIPTOR(ts), INTERVAL '5' SECOND, INTERVAL '10' SECOND)
)
GROUP BY window_start, window_end, user_id;

-- 累积窗口 TVF
SELECT
    window_start, window_end, user_id,
    COUNT(url) AS cnt
FROM TABLE(
    CUMULATE(TABLE kafka_source, DESCRIPTOR(ts), INTERVAL '5' SECOND, INTERVAL '1' HOUR)
)
GROUP BY window_start, window_end, user_id;
```

### 10.5 ⭐ Top-N 语法

```sql
-- 每小时 URL 访问量 Top 3
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (
        PARTITION BY window_start, window_end
        ORDER BY cnt DESC
    ) AS row_num
    FROM (
        SELECT
            window_start, window_end, url,
            COUNT(*) AS cnt
        FROM TABLE(
            TUMBLE(TABLE kafka_source, DESCRIPTOR(ts), INTERVAL '1' HOUR)
        )
        GROUP BY window_start, window_end, url
    )
) WHERE row_num <= 3;
```

### 10.6 ⭐ Deduplication 去重

```sql
-- 对 user_id 去重，保留第一条
SELECT user_id, url, ts FROM (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY user_id ORDER BY ts ASC
        ) AS row_num
    FROM kafka_source
) WHERE row_num = 1;
```

### 10.7 ⭐ 代码中使用 Flink SQL

```java
// 创建表环境
StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

// 从 DataStream 创建表
Table table = tableEnv.fromDataStream(stream, $("user"), $("url"), $("ts").rowtime());

// 注册临时视图
tableEnv.createTemporaryView("clicks", table);

// 执行 SQL
Table result = tableEnv.sqlQuery(
    "SELECT user, COUNT(url) AS cnt FROM clicks GROUP BY user"
);

// 转回 DataStream
tableEnv.toChangelogStream(result).print();
```

---

## 🔥 Flink 面试高频问题

### Q1：Flink 的核心组件和作业提交流程？

**核心组件**：Client → JobManager（Dispatcher + ResourceManager + JobMaster） → TaskManager

**YARN Application 模式提交流程**：
1. Client 将作业提交到 YARN
2. YARN 启动 ApplicationMaster（包含 JobManager）
3. JobManager 中的 JobMaster 解析 JobGraph → ExecutionGraph
4. ResourceManager 向 YARN 申请资源（Container）
5. YARN 启动 TaskManager
6. TaskManager 注册到 ResourceManager
7. ResourceManager 分配 Slot 给 JobMaster
8. JobMaster 将 Task 部署到 TaskManager 的 Slot 中

### Q2：Flink 如何实现精确一次（Exactly-Once）？

三层保障：
1. **输入端**：Kafka 支持 Offset 重放
2. **Flink 内部**：Checkpoint 机制（Chandy-Lamport 算法 + Barrier 对齐）
3. **输出端**：两阶段提交（2PC），预提交 → Checkpoint 完成 → 正式提交

🔥 **关键配置**：
- `env.enableCheckpointing(5000L, CheckpointingMode.EXACTLY_ONCE)`
- Kafka Sink 设置 `DeliveryGuarantee.EXACTLY_ONCE`
- Kafka Broker `transaction.max.timeout.ms` 需大于 Checkpoint 超时时间

### Q3：Flink Checkpoint 和 Spark Checkpoint 的区别？

| 对比 | Flink Checkpoint | Spark Checkpoint |
|------|-----------------|-----------------|
| **触发方式** | 🔥 自动周期性触发 | 手动调用 |
| **存储内容** | 算子状态 + Kafka Offset | RDD 数据 |
| **一致性** | 精确一次 | 至少一次 |
| **恢复方式** | 从快照恢复状态 | 重新计算 RDD |
| **性能影响** | 异步 + 增量，影响小 | 需要额外计算 |

### Q4：Flink 的 Watermark 机制和迟到数据处理？

**Watermark 本质**：衡量事件时间进展的标记
- 单调递增，表示"不会再有 ≤ 该时间戳的数据到来"
- **公式**：`Watermark = maxEventTime - maxOutOfOrderness - 1ms`

**迟到数据三重保障**：
1. **Watermark 延迟**：`forBoundedOutOfOrderness(Duration.ofSeconds(5))`
2. **窗口允许迟到**：`.allowedLateness(Time.minutes(1))`
3. **侧输出流兜底**：`.sideOutputLateData(lateTag)`

**多并行度水位线传递**：取上游所有通道水位线的**最小值**

### Q5：Flink 内存模型？

**TaskManager 内存结构**：
- **JVM 堆内存**
  - Framework Heap（框架堆内存，128MB）
  - 🔥 Task Heap（任务堆内存，用户代码使用）
- **JVM 堆外内存**
  - Framework Off-Heap（框架堆外，32MB）
  - Task Off-Heap（任务堆外）
  - 🔥 Managed Memory（托管内存，RocksDB/排序/哈希表使用，默认 40%）
  - Network Memory（网络缓冲，默认 10%）
- **JVM Metaspace**（256MB）
- **JVM Overhead**（JVM 开销，默认 10%）

### Q6：Flink 反压（Backpressure）如何处理？

**定位**：Flink Web UI → Task → Back Pressure → 观察 HIGH/OK

**解决方案**：
1. 🔥 **提高并行度**：增加算子并行度
2. 🔥 **优化算子逻辑**：减少序列化、避免阻塞操作
3. **调整 Buffer 大小**：增大 `taskmanager.network.memory.fraction`
4. **异步 I/O**：使用 `AsyncDataStream` 处理外部调用
5. **开启非对齐 Checkpoint**：减少 Barrier 等待时间
6. **数据倾斜处理**：rebalance、随机 Key 打散

### Q7：Flink 状态管理和状态后端选择？

**状态分类**：
- **Keyed State**：按 Key 隔离，包括 ValueState、ListState、MapState 等
- **Operator State**：绑定到算子实例，如 Kafka Offset

**状态后端选择**：
- **HashMapStateBackend**：状态在 JVM 堆内存，读写快，适合状态小的场景
- 🔥 **EmbeddedRocksDBStateBackend**：状态在磁盘，支持增量 Checkpoint，**生产推荐**

### Q8：Flink 数据倾斜如何解决？

1. 🔥 **rebalance()**：强制轮询重分区
2. 🔥 **随机 Key 前缀 + 二次聚合**：先打散预聚合，再去掉前缀最终聚合
3. **Broadcast Join**：小表广播，避免 Shuffle
4. **调整并行度**：增加下游算子并行度
5. **AQE 自适应优化**（Flink SQL）
6. **侧输出流隔离热点 Key**：单独处理倾斜数据

### Q9：Flink 常见的维表 Join 方案？

| 方案 | 说明 | 优缺点 |
|------|------|--------|
| **预加载维表** | 在 open() 中加载到内存 | 简单，但不支持更新 |
| **热存储关联** | 查询 Redis/HBase | 🔥 实时性好，有网络开销 |
| **广播维表** | 广播流 + Connect | 支持更新，内存消耗大 |
| **Temporal Join** | Flink SQL 时态表 Join | 🔥 语义最完整 |
| **Lookup Join** | Flink SQL `FOR SYSTEM_TIME AS OF` | 🔥 **推荐**，支持异步+缓存 |

### Q10：Flink on YARN 的部署模式区别？

| 模式 | Driver 位置 | 集群生命周期 | 资源隔离 | 推荐场景 |
|------|------------|-------------|---------|----------|
| **Session** | Client | 常驻 | ❌ 共享 | 开发测试 |
| **Per-Job** | Client | 作业独立 | ✅ | Flink <1.15 |
| **🔥 Application** | JobManager | 作业独立 | ✅ | **生产推荐** |
