# Kafka

> 🔗 **官方网站**：https://kafka.apache.org/
> 📖 **官方文档**：https://kafka.apache.org/documentation/
> 📌 **学习版本**：Kafka 3.3.x

---

## 第1章 Kafka 概述 ⭐

### 1.1 什么是 Kafka

Kafka 是一个**分布式流处理平台**，最初由 LinkedIn 开发，后成为 Apache 顶级项目。主要用途：
1. **消息队列**：应用解耦、异步处理、削峰填谷
2. **流处理**：实时数据流处理管道
3. **数据集成**：连接不同数据系统的中间件

### 1.2 Kafka 架构 🔥

![Kafka架构](https://kafka.apache.org/images/kafka_diagram.png ':size=700')

| 核心组件 | 说明 |
|:---|:---|
| **Producer（生产者）** | 向 Kafka Topic 发送消息 |
| **Consumer（消费者）** | 从 Topic 消费消息 |
| **Broker（服务节点）** | Kafka 集群中的每个服务器节点 |
| **Topic（主题）** | 消息的分类，逻辑概念 |
| **Partition（分区）** | Topic 的物理划分，**保证分区内有序** |
| **Replica（副本）** | 分区的备份，Leader + Follower |
| **Consumer Group** | 消费者组，组内消费者共同消费Topic |
| **ZooKeeper / KRaft** | 集群元数据管理（3.x 开始支持 KRaft 去 ZK） |

> 💡 **面试问法**：请描述 Kafka 的架构？
> **答**：Kafka 集群由多个 Broker 组成，消息按 Topic 分类，每个 Topic 分为多个 Partition 实现并行，Partition 有多个 Replica 实现高可用。Producer 发消息到 Partition，Consumer Group 中的 Consumer 分配 Partition 进行消费。

### 1.3 核心概念详解 🔥

#### Partition 与 Offset

```
Topic: order-topic (3 partitions)
┌────────────────────────────────┐
│ Partition 0: [0][1][2][3][4]   │  ← offset
│ Partition 1: [0][1][2][3]      │
│ Partition 2: [0][1][2][3][4][5]│
└────────────────────────────────┘
```

- 每个 Partition 是一个**有序、不可变**的消息序列
- **Offset**：消息在分区内的唯一编号，**分区内有序，全局无序**
- 消费者通过 Offset 追踪消费进度

#### 副本机制

| 角色 | 说明 |
|:---|:---|
| **Leader** | 处理所有读写请求 |
| **Follower** | 从 Leader 同步数据，不直接服务读写 |
| **ISR** | In-Sync Replicas，与 Leader 保持同步的副本集合 |
| **OSR** | Out-of-Sync Replicas，落后的副本 |

---

## 第2章 安装部署 📝

### 2.1 关键配置

```properties
# server.properties 核心配置
broker.id=0                                    # 每个broker唯一
listeners=PLAINTEXT://hadoop102:9092          # 监听地址
log.dirs=/opt/module/kafka/datas              # 数据目录
num.partitions=3                              # 默认分区数
default.replication.factor=2                  # 默认副本数
log.retention.hours=168                       # 数据保留7天
zookeeper.connect=hadoop102:2181,hadoop103:2181,hadoop104:2181/kafka
```

### 2.2 常用命令

```bash
# 启动 Kafka
bin/kafka-server-start.sh -daemon config/server.properties

# Topic 操作
bin/kafka-topics.sh --bootstrap-server hadoop102:9092 --create --topic first --partitions 3 --replication-factor 2
bin/kafka-topics.sh --bootstrap-server hadoop102:9092 --list
bin/kafka-topics.sh --bootstrap-server hadoop102:9092 --describe --topic first

# 生产者
bin/kafka-console-producer.sh --bootstrap-server hadoop102:9092 --topic first

# 消费者
bin/kafka-console-consumer.sh --bootstrap-server hadoop102:9092 --topic first --from-beginning
bin/kafka-console-consumer.sh --bootstrap-server hadoop102:9092 --topic first --group my-group
```

---

## 第3章 Producer 生产者 🔥

### 3.1 发送流程 🔥🔥

```
Producer 发送流程：
┌──────────┐   ┌──────────────┐   ┌───────────┐   ┌────────┐
│ main线程  │──▶│ Interceptor  │──▶│ Serializer│──▶│Partitioner│
└──────────┘   └──────────────┘   └───────────┘   └────────┘
                                                       │
                                          ┌────────────▼─────────────┐
                                          │  RecordAccumulator 缓冲区 │
                                          │  (默认32MB，batch=16KB)    │
                                          └────────────┬─────────────┘
                                                       │
                                                 ┌─────▼─────┐
                                                 │ Sender线程 │
                                                 │(max.in.flight│
                                                 │.requests = 5)│
                                                 └─────┬─────┘
                                                       │
                                                 ┌─────▼─────┐
                                                 │  Broker    │
                                                 └───────────┘
```

### 3.2 分区策略 🔥

| 场景 | 策略 |
|:---|:---|
| 指定 partition | 直接使用指定值 |
| 未指定 partition，有 key | `hash(key) % numPartitions` |
| 无 partition 无 key | **粘性分区**（Sticky Partition，同一 batch 发到同一分区） |

### 3.3 Java API 🔥

```java
import org.apache.kafka.clients.producer.*;
import java.util.Properties;

public class KafkaProducerExample {
    public static void main(String[] args) {
        // 1. 配置
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "hadoop102:9092,hadoop103:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
        
        // 重要参数
        props.put(ProducerConfig.ACKS_CONFIG, "all");           // acks=-1 最高可靠
        props.put(ProducerConfig.RETRIES_CONFIG, 3);             // 重试次数
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);      // batch大小 16KB
        props.put(ProducerConfig.LINGER_MS_CONFIG, 5);           // 等待时间 5ms
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432); // 缓冲区 32MB
        
        // 2. 创建 Producer
        KafkaProducer<String, String> producer = new KafkaProducer<>(props);
        
        // 3. 发送消息
        for (int i = 0; i < 10; i++) {
            // 异步发送
            producer.send(new ProducerRecord<>("first", "key" + i, "value" + i), 
                (metadata, exception) -> {
                    if (exception == null) {
                        System.out.println("Topic: " + metadata.topic() + 
                            ", Partition: " + metadata.partition() + 
                            ", Offset: " + metadata.offset());
                    } else {
                        exception.printStackTrace();
                    }
                });
        }
        
        // 4. 关闭
        producer.close();
    }
}
```

### 3.4 ACKs 机制 🔥🔥

| acks | 含义 | 可靠性 | 吞吐量 | 适用场景 |
|:---|:---|:---|:---|:---|
| `0` | 不等确认 | 最低 | **最高** | 日志收集（允许丢数据）|
| `1` | Leader 确认 | 中 | 中 | 一般场景 |
| `-1(all)` | ISR 全部确认 | **最高** | 最低 | 金融/订单（不允许丢数据）|

> 💡 **面试问法**：如何保证 Kafka 数据不丢失？
> **答**：`acks = -1` + `min.insync.replicas >= 2` + `replication.factor >= 3`。生产者端：acks=all + 重试。Broker 端：副本数≥3，ISR≥2。消费者端：手动提交 Offset。

### 3.5 幂等性和事务 🔥

```java
// 开启幂等性（默认开启）
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
// 原理：ProducerID + SequenceNumber 去重

// 事务（精确一次语义）
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "my-transactional-id");

producer.initTransactions();
try {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("topic1", "msg1"));
    producer.send(new ProducerRecord<>("topic2", "msg2"));
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

---

## 第4章 Broker 核心机制 🔥

### 4.1 日志存储结构 🔥

```
kafka-logs/
├── first-0/                     # Topic=first, Partition=0
│   ├── 00000000000000000000.log  # 数据文件（1GB 一个 Segment）
│   ├── 00000000000000000000.index  # 偏移量索引（稀疏索引）
│   ├── 00000000000000000000.timeindex  # 时间戳索引
│   └── leader-epoch-checkpoint
├── first-1/
└── first-2/
```

### 4.2 数据清理策略 ⭐

| 策略 | 参数 | 说明 |
|:---|:---|:---|
| **删除（Delete）** | `log.retention.hours=168` | 默认保留7天 |
| **压缩（Compact）** | `log.cleanup.policy=compact` | 保留每个 key 的最新值 |

### 4.3 Leader 选举 🔥

1. Controller 在 ZK 注册并监听 Broker 变化
2. 某个 Broker 宕机，Controller 感知
3. 从 ISR 列表中选择新 Leader（优先选 ISR 中第一个）
4. 更新 Leader 信息并通知所有 Broker

### 4.4 副本同步机制 🔥

| 概念 | 说明 |
|:---|:---|
| **LEO** | Log End Offset，每个副本最后一条消息的 offset+1 |
| **HW** | High Watermark，ISR 中最小的 LEO，消费者只能看到 HW 之前的消息 |

```
Leader:   [0][1][2][3][4][5]  LEO=6
                         ↑ HW=4
Follower1:[0][1][2][3]        LEO=4
Follower2:[0][1][2][3][4]     LEO=5

消费者只能消费 offset 0~3 的消息（HW之前）
```

---

## 第5章 Consumer 消费者 🔥

### 5.1 消费者组 🔥🔥

- **消费者组**内的消费者**共同消费** Topic 的分区
- 一个分区只能被组内的**一个消费者**消费
- 不同消费者组可以**独立消费**同一 Topic

```
Topic: first (3 partitions)
┌─────────────────────────────────┐
│ Consumer Group A                 │
│ ┌──────┐ ┌──────┐ ┌──────┐     │
│ │ C1   │ │ C2   │ │ C3   │     │
│ │ P0   │ │ P1   │ │ P2   │     │
│ └──────┘ └──────┘ └──────┘     │
└─────────────────────────────────┘
```

### 5.2 消费方式

```java
import org.apache.kafka.clients.consumer.*;
import java.time.Duration;
import java.util.*;

public class KafkaConsumerExample {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "hadoop102:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "test-group");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");  // 从头消费
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");    // 关闭自动提交

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Arrays.asList("first"));

        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
            for (ConsumerRecord<String, String> record : records) {
                System.out.printf("Topic=%s, Partition=%d, Offset=%d, Key=%s, Value=%s%n",
                    record.topic(), record.partition(), record.offset(), record.key(), record.value());
            }
            // 手动同步提交 Offset
            consumer.commitSync();
        }
    }
}
```

### 5.3 分区分配策略 🔥

| 策略 | 说明 | 适用场景 |
|:---|:---|:---|
| **Range** | 按分区号范围分配 | 默认策略 |
| **RoundRobin** | 轮询分配 | 多 Topic 场景 |
| **Sticky** | 粘性分配，尽量保持上次分配 | **推荐**，减少 Rebalance |
| **CooperativeSticky** | 协作粘性，增量 Rebalance | Kafka 2.4+ |

### 5.4 Offset 管理 🔥

```java
// 手动提交 Offset
consumer.commitSync();      // 同步提交（阻塞）
consumer.commitAsync();     // 异步提交（非阻塞）

// 指定 Offset 消费
TopicPartition tp = new TopicPartition("first", 0);
consumer.assign(Arrays.asList(tp));
consumer.seek(tp, 100);  // 从 offset=100 开始消费

// 指定时间消费
Map<TopicPartition, Long> timestampMap = new HashMap<>();
timestampMap.put(tp, System.currentTimeMillis() - 24 * 3600 * 1000); // 一天前
Map<TopicPartition, OffsetAndTimestamp> offsets = consumer.offsetsForTimes(timestampMap);
```

---

## 第6章 Kafka 生产调优 🔥

### 6.1 吞吐量优化

```properties
# Producer 端
batch.size=32768              # 增大 batch 到 32KB
linger.ms=5                   # 等待 5ms
buffer.memory=67108864        # 缓冲区 64MB
compression.type=snappy       # 启用压缩

# Broker 端
num.replica.fetchers=3        # 增加副本拉取线程
num.network.threads=6
num.io.threads=16

# Consumer 端
fetch.max.bytes=52428800      # 50MB
max.poll.records=500
```

### 6.2 可靠性配置

```properties
# 不允许丢数据的黄金配置
acks=all
min.insync.replicas=2
replication.factor=3
enable.idempotence=true
retries=2147483647
max.in.flight.requests.per.connection=5
```

### 6.3 消息积压处理 🔥

| 原因 | 解决方案 |
|:---|:---|
| 消费能力不足 | 增加 Consumer 数量（不超过分区数） |
| 下游处理慢 | 优化下游逻辑 / 异步处理 |
| 单条消息处理耗时 | 批量处理 / 增加 `max.poll.records` |
| 分区数不够 | 增加 Topic 分区数 |

---

## 第7章 高频面试题 🔥🔥🔥

### Q1：Kafka 为什么这么快？
> 1. **顺序读写磁盘**（比随机读写快 600 倍）
> 2. **零拷贝**（Zero-Copy，sendfile 系统调用）
> 3. **分区并行**（多 Partition 并行读写）
> 4. **批量发送 + 压缩**
> 5. **Page Cache**（利用操作系统页缓存）

### Q2：如何保证消息不丢失？
> - **Producer**：acks=all + 重试 + 幂等性
> - **Broker**：replication.factor≥3，min.insync.replicas≥2
> - **Consumer**：手动提交 Offset，处理完再提交

### Q3：如何保证消息不重复消费？
> 1. 开启幂等性（enable.idempotence=true）
> 2. 消费端做**幂等处理**（数据库唯一键 / Redis Set 去重）
> 3. 开启事务

### Q4：如何保证消息有序？
> - **分区内有序**：同一 key 的消息发到同一分区
> - 设置 `max.in.flight.requests.per.connection=1`（牺牲吞吐量）
> - 开启幂等性后可以设为 5（Kafka 保证有序）

### Q5：Kafka 和 RabbitMQ、RocketMQ 的区别？
> | 对比 | Kafka | RabbitMQ | RocketMQ |
> |:---|:---|:---|:---|
> | 吞吐量 | **百万级** | 万级 | 十万级 |
> | 延迟 | ms 级 | μs 级 | ms 级 |
> | 消息可靠 | 高（副本+ISR） | 高（镜像队列） | 高 |
> | 协议 | 自定义 | AMQP | 自定义 |
> | 定位 | 流处理平台 | 企业消息 | 电商场景 |

### Q6：ISR、OSR、AR 分别代表什么？
> - **AR** = All Replicas（所有副本）
> - **ISR** = In-Sync Replicas（同步副本集合）
> - **OSR** = Out-of-Sync Replicas（落后副本）
> - AR = ISR + OSR
