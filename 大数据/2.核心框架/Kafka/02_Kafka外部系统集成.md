# Kafka 外部系统集成

> 🔗 **官方文档**：https://kafka.apache.org/documentation/#connect
> 📌 **学习版本**：Kafka 3.3.x

---

## 第1章 集成 Flume 🔥

### 1.1 Flume → Kafka（Flume 作为 Producer）

```properties
# flume-kafka-sink.conf
a1.sources = r1
a1.channels = c1
a1.sinks = k1

# Source: 监控文件
a1.sources.r1.type = TAILDIR
a1.sources.r1.filegroups = f1
a1.sources.r1.filegroups.f1 = /opt/module/applog/log/app.*
a1.sources.r1.positionFile = /opt/module/flume/taildir_position.json

# Channel: Kafka Channel（推荐，跳过 Sink 直接写 Kafka）
a1.channels.c1.type = org.apache.flume.channel.kafka.KafkaChannel
a1.channels.c1.kafka.bootstrap.servers = hadoop102:9092,hadoop103:9092
a1.channels.c1.kafka.topic = topic_log
a1.channels.c1.parseAsFlumeEvent = false

a1.sources.r1.channels = c1
```

> 💡 **生产最佳实践**：使用 **KafkaChannel** 而非 KafkaSink，省去 Channel 到 Sink 的步骤，效率更高。

### 1.2 Kafka → Flume（Flume 作为 Consumer）

```properties
# kafka-flume-hdfs.conf
a1.sources = r1
a1.channels = c1
a1.sinks = k1

# Source: Kafka Source
a1.sources.r1.type = org.apache.flume.source.kafka.KafkaSource
a1.sources.r1.kafka.bootstrap.servers = hadoop102:9092
a1.sources.r1.kafka.topics = topic_log
a1.sources.r1.kafka.consumer.group.id = flume-consumer-group

# Channel: File Channel
a1.channels.c1.type = file
a1.channels.c1.dataDirs = /opt/module/flume/data
a1.channels.c1.checkpointDir = /opt/module/flume/checkpoint

# Sink: HDFS
a1.sinks.k1.type = hdfs
a1.sinks.k1.hdfs.path = hdfs://hadoop102:8020/origin_data/log/%Y-%m-%d
a1.sinks.k1.hdfs.filePrefix = log
a1.sinks.k1.hdfs.rollInterval = 10
a1.sinks.k1.hdfs.rollSize = 134217728
a1.sinks.k1.hdfs.rollCount = 0
a1.sinks.k1.hdfs.fileType = CompressedStream
a1.sinks.k1.hdfs.codeC = gzip

a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

---

## 第2章 集成 Flink 🔥🔥

### 2.1 Flink 消费 Kafka

```java
// FlinkKafkaConsumer (Flink 1.x~1.14)
// KafkaSource (Flink 1.14+)
KafkaSource<String> source = KafkaSource.<String>builder()
    .setBootstrapServers("hadoop102:9092")
    .setTopics("topic_log")
    .setGroupId("flink-consumer-group")
    .setStartingOffsets(OffsetsInitializer.latest())
    .setValueOnlyDeserializer(new SimpleStringSchema())
    .build();

DataStream<String> stream = env.fromSource(
    source, WatermarkStrategy.noWatermarks(), "Kafka Source");
```

### 2.2 Flink 写入 Kafka

```java
KafkaSink<String> sink = KafkaSink.<String>builder()
    .setBootstrapServers("hadoop102:9092")
    .setRecordSerializer(
        KafkaRecordSerializationSchema.builder()
            .setTopic("topic_output")
            .setValueSerializationSchema(new SimpleStringSchema())
            .build()
    )
    .setDeliveryGuarantee(DeliveryGuarantee.EXACTLY_ONCE)
    .build();

stream.sinkTo(sink);
```

---

## 第3章 集成 SpringBoot ⭐

### 3.1 依赖配置

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

### 3.2 生产者

```java
@RestController
public class KafkaProducerController {
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @GetMapping("/send")
    public String send(@RequestParam String msg) {
        kafkaTemplate.send("test-topic", msg);
        return "success";
    }
}
```

### 3.3 消费者

```java
@Component
public class KafkaConsumerListener {
    @KafkaListener(topics = "test-topic", groupId = "spring-group")
    public void listen(String message) {
        System.out.println("收到消息: " + message);
    }
}
```

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: hadoop102:9092,hadoop103:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
    consumer:
      group-id: spring-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      auto-offset-reset: latest
```

---

## 第4章 集成 Spark ⭐

### 4.1 Spark Streaming 消费 Kafka

```scala
val kafkaParams = Map[String, Object](
    "bootstrap.servers" -> "hadoop102:9092",
    "group.id" -> "spark-kafka-group",
    "key.deserializer" -> classOf[StringDeserializer],
    "value.deserializer" -> classOf[StringDeserializer],
    "auto.offset.reset" -> "latest",
    "enable.auto.commit" -> (false: java.lang.Boolean)
)

val stream = KafkaUtils.createDirectStream[String, String](
    ssc,
    LocationStrategies.PreferConsistent,
    ConsumerStrategies.Subscribe[String, String](
        Array("topic_log"), kafkaParams)
)

// 手动提交 Offset
stream.foreachRDD { rdd =>
    val offsets = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
    // 处理数据 ...
    stream.asInstanceOf[CanCommitOffsets].commitAsync(offsets)
}
```

### 4.2 Structured Streaming 消费 Kafka

```scala
val df = spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "hadoop102:9092")
    .option("subscribe", "topic_log")
    .option("startingOffsets", "latest")
    .load()
    .selectExpr("CAST(value AS STRING)")
```

---

## 🔥 面试高频题

### Q1：Flume 和 Kafka 如何集成？
> 两种方式：1）Flume 作为 Producer，使用 KafkaChannel（推荐）或 KafkaSink 将数据写入 Kafka；2）Flume 作为 Consumer，使用 KafkaSource 从 Kafka 消费数据写入 HDFS。

### Q2：Flink 如何保证 Kafka 端到端 Exactly-Once？
> Flink 使用 Kafka 的事务机制 + Checkpoint：Producer 设置 `DeliveryGuarantee.EXACTLY_ONCE`，Consumer 设置 `isolation.level=read_committed`，配合 Flink 的两阶段提交协议实现。
