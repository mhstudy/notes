# Spark Streaming

> 🔗 **官方文档**：https://spark.apache.org/docs/latest/streaming-programming-guide.html
> 📌 **学习版本**：Spark 3.x
> ⚠️ 注意：Spark Streaming 已进入维护模式，推荐使用 **Structured Streaming**

---

## 第1章 Spark Streaming 概述 ⭐

### 1.1 什么是 Spark Streaming

Spark Streaming 是 Spark 的**准实时**（微批处理）流计算框架，将实时数据流按时间间隔切分为小批次（micro-batch），用 Spark Core 引擎处理。

### 1.2 Spark Streaming 架构 🔥

```
数据源(Kafka/Socket/Flume)
       │
       ▼ (接收数据)
┌─────────────────────────────┐
│     Spark Streaming         │
│                             │
│  实时数据流 → 按时间切分      │
│       │                     │
│       ▼                     │
│  ┌─────┐ ┌─────┐ ┌─────┐  │
│  │batch│ │batch│ │batch│   │  ← DStream (离散化流)
│  │ t0  │ │ t1  │ │ t2  │   │
│  └──┬──┘ └──┬──┘ └──┬──┘  │
│     │       │       │      │
│     ▼       ▼       ▼      │
│    RDD     RDD     RDD     │  ← 每个 batch 就是一个 RDD
│  (Spark Core 引擎处理)      │
└─────────────────────────────┘
       │
       ▼
  输出结果(HDFS/DB/Kafka)
```

### 1.3 DStream 与 RDD 的关系

```
DStream 本质上是一系列 RDD 的序列:
DStream = RDD@t0 + RDD@t1 + RDD@t2 + ...

对 DStream 的操作最终会转化为对每个 RDD 的操作。
```

### 1.4 背压机制（Back Pressure）⭐

```
问题: 数据产生速度 > 消费速度 → 内存溢出
解决: 开启背压机制，自动调节接收速率

spark.streaming.backpressure.enabled = true   // 开启背压
spark.streaming.receiver.maxRate = 100        // 接收器最大速率(条/秒)
spark.streaming.kafka.maxRatePerPartition = 100  // Kafka 每分区最大速率
```

---

## 第2章 DStream 入门 🔥

### 2.1 WordCount（Socket 版）

```scala
import org.apache.spark.SparkConf
import org.apache.spark.streaming.{Seconds, StreamingContext}

object StreamWordCount {
    def main(args: Array[String]): Unit = {
        // 1. 创建 SparkConf
        val conf = new SparkConf().setMaster("local[*]").setAppName("StreamWordCount")

        // 2. 创建 StreamingContext（批次间隔 3 秒）
        val ssc = new StreamingContext(conf, Seconds(3))

        // 3. 从 Socket 接收数据
        val lineStream = ssc.socketTextStream("hadoop102", 9999)

        // 4. 处理数据
        val wordCount = lineStream
            .flatMap(_.split(" "))
            .map((_, 1))
            .reduceByKey(_ + _)

        // 5. 输出
        wordCount.print()

        // 6. 启动
        ssc.start()
        ssc.awaitTermination()
    }
}
```

```bash
# 测试：开启 Socket 服务
nc -lk 9999
# 输入数据
hello spark hello streaming
```

---

## 第3章 DStream 数据源 🔥

### 3.1 Kafka 数据源（开发重点）🔥🔥

```scala
import org.apache.kafka.clients.consumer.ConsumerConfig
import org.apache.spark.streaming.kafka010._

object KafkaStreamDemo {
    def main(args: Array[String]): Unit = {
        val conf = new SparkConf().setMaster("local[*]").setAppName("KafkaStream")
        val ssc = new StreamingContext(conf, Seconds(3))

        // Kafka 参数
        val kafkaParams = Map[String, Object](
            ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG -> "hadoop102:9092,hadoop103:9092",
            ConsumerConfig.GROUP_ID_CONFIG -> "spark-streaming-group",
            ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG ->
                "org.apache.kafka.common.serialization.StringDeserializer",
            ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG ->
                "org.apache.kafka.common.serialization.StringDeserializer"
        )

        // 消费 Kafka 数据
        val kafkaStream = KafkaUtils.createDirectStream[String, String](
            ssc,
            LocationStrategies.PreferConsistent,
            ConsumerStrategies.Subscribe[String, String](
                Array("spark-topic"), kafkaParams
            )
        )

        // 处理数据
        kafkaStream
            .map(_.value())
            .flatMap(_.split(" "))
            .map((_, 1))
            .reduceByKey(_ + _)
            .print()

        ssc.start()
        ssc.awaitTermination()
    }
}
```

---

## 第4章 DStream 转换 🔥

### 4.1 无状态转换

和 RDD 类似的算子：`map`、`flatMap`、`filter`、`reduceByKey`、`join` 等。

### 4.2 有状态转换

#### UpdateStateByKey（全局状态累加）🔥

```scala
// 全局 WordCount（累计计数）
ssc.checkpoint("hdfs://hadoop102:8020/checkpoint")

val wordCount = lineStream
    .flatMap(_.split(" "))
    .map((_, 1))
    .updateStateByKey[Int] { (values: Seq[Int], state: Option[Int]) =>
        val currentCount = values.sum
        val previousCount = state.getOrElse(0)
        Some(currentCount + previousCount)
    }
```

> ⚠️ 使用 `updateStateByKey` 必须设置 **Checkpoint**。

#### Window 窗口操作 🔥🔥

```scala
// 窗口参数:
// windowDuration: 窗口长度（必须是批次间隔的整数倍）
// slideDuration:  滑动间隔（必须是批次间隔的整数倍）

// 示例: 每 6 秒统计最近 12 秒的 WordCount
val windowedWordCount = wordStream
    .reduceByKeyAndWindow(
        (a: Int, b: Int) => a + b,   // 窗口加入时的聚合
        (a: Int, b: Int) => a - b,   // 窗口滑出时的反向操作（优化版）
        Seconds(12),                  // 窗口长度
        Seconds(6)                    // 滑动间隔
    )
```

```
窗口滑动示意:
时间: ──1──2──3──4──5──6──7──8──9──10──11──12──→
窗口1: [───────────────────────────]
窗口2:          [───────────────────────────]
                     12秒窗口，6秒滑动
```

---

## 第5章 DStream 输出 ⭐

```scala
// 输出到控制台
dstream.print()

// 保存为文本文件
dstream.saveAsTextFiles("hdfs://hadoop102:8020/output/prefix")

// 自定义输出（如写入 MySQL）
dstream.foreachRDD { rdd =>
    rdd.foreachPartition { iter =>
        val conn = DriverManager.getConnection(url, user, password)
        iter.foreach { record =>
            val sql = "INSERT INTO result VALUES (?, ?)"
            val ps = conn.prepareStatement(sql)
            ps.setString(1, record._1)
            ps.setInt(2, record._2)
            ps.executeUpdate()
        }
        conn.close()
    }
}
```

> 💡 **最佳实践**：使用 `foreachPartition` 而非 `foreach`，避免频繁创建连接。

---

## 第6章 优雅关闭 ⭐

```scala
// 方式一：设置优雅关闭
conf.set("spark.streaming.stopGracefullyOnShutdown", "true")

// 方式二：第三方信号（如 HDFS 文件标记）
new Thread(() => {
    while (true) {
        Thread.sleep(5000)
        val fs = FileSystem.get(new URI("hdfs://hadoop102:8020"), new Configuration())
        if (fs.exists(new Path("/stop_streaming"))) {
            ssc.stop(stopSparkContext = true, stopGracefully = true)
            System.exit(0)
        }
    }
}).start()
```

---

## 第7章 Structured Streaming（推荐）🔥🔥

Spark 2.x 后推出的**新一代流处理 API**，基于 DataFrame/DataSet，取代 Spark Streaming。

```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession.builder()
    .appName("StructuredStreaming")
    .master("local[*]")
    .getOrCreate()

import spark.implicits._

// 从 Kafka 读取流
val kafkaDF = spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "hadoop102:9092")
    .option("subscribe", "spark-topic")
    .load()

// 处理
val wordCount = kafkaDF
    .selectExpr("CAST(value AS STRING)")
    .as[String]
    .flatMap(_.split(" "))
    .groupBy("value")
    .count()

// 输出到控制台
val query = wordCount.writeStream
    .outputMode("complete")
    .format("console")
    .start()

query.awaitTermination()
```

---

## 🔥 面试高频题

### Q1：Spark Streaming 和 Flink 的区别？
> Spark Streaming 是**微批处理**（准实时），以时间间隔将数据切分为 RDD 处理；Flink 是**真正的流处理**（逐条处理），延迟更低。Spark 吞吐量高，Flink 延迟低。

### Q2：DStream 的 Window 操作原理？
> 窗口操作将多个批次的 RDD 合并计算。窗口长度和滑动间隔必须是批次间隔的整数倍。优化版 `reduceByKeyAndWindow` 通过反向操作复用上个窗口的计算结果。

### Q3：Spark Streaming 如何保证 Exactly-Once？
> 1）接收端：使用 Direct API 从 Kafka 消费（自己管理 Offset）；2）处理端：Checkpoint + 幂等写入或事务写入；3）输出端：幂等操作或事务提交（如写入支持事务的数据库）。
