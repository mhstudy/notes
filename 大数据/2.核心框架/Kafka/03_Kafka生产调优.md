# Kafka 生产调优

> 🔗 **官方文档**：https://kafka.apache.org/documentation/
> 📌 **学习版本**：Kafka 3.3.x

---

## 第1章 Kafka 硬件配置选择 📝

### 1.1 服务器台数选择

```
公式: 服务器台数 = 2 × (峰值生产速度 × 副本数 / 100) + 1

示例: 峰值 20MB/s, 3副本
     = 2 × (20 × 3 / 100) + 1 = 2.2 → 3台
```

### 1.2 硬件选择

| 硬件 | 建议 |
|:---|:---|
| **磁盘** | 机械硬盘即可（顺序读写，SSD 提升不大） |
| **内存** | Kafka 占 JVM 堆 6~10GB + 页缓存（越大越好） |
| **CPU** | 32核+（num.io.threads + num.network.threads + 其他线程） |
| **网络** | 千兆网卡是底线，万兆更好 |

---

## 第2章 Kafka 生产者调优 🔥

### 2.1 核心参数 🔥🔥

| 参数 | 默认值 | 建议 | 说明 |
|:---|:---|:---|:---|
| `batch.size` | 16KB | **32KB** | 批次大小（越大吞吐越高） |
| `linger.ms` | 0 | **5~100** | 等待时间（等够批次再发） |
| `buffer.memory` | 32MB | **64MB** | 缓冲区总大小 |
| `compression.type` | none | **snappy/lz4** | 压缩类型 |
| `acks` | -1 | 按需 | 应答级别 |
| `retries` | MAX | 保持 | 重试次数 |
| `max.in.flight.requests.per.connection` | 5 | ≤5 | 幂等开启时不超过5 |

### 2.2 提高吞吐量 🔥

```java
Properties props = new Properties();
// 1. 增大批次大小
props.put(ProducerConfig.BATCH_SIZE_CONFIG, 32768);  // 32KB
// 2. 增加等待时间（凑满批次）
props.put(ProducerConfig.LINGER_MS_CONFIG, 5);
// 3. 开启压缩
props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
// 4. 增大缓冲区
props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 67108864);  // 64MB
```

### 2.3 数据可靠性 🔥🔥

| acks | 说明 | 数据可靠性 | 吞吐量 |
|:---|:---|:---|:---|
| **0** | 不等应答 | 最低（可能丢数据） | 最高 |
| **1** | Leader 写入即应答 | 中等 | 中等 |
| **-1(all)** | ISR 全部写入才应答 | **最高** | 最低 |

> 💡 生产环境推荐：**acks=-1 + min.insync.replicas=2**

### 2.4 数据去重 🔥🔥

#### 幂等性（Exactly Once 语义）

```java
// 开启幂等性
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
// 自动设置: acks=-1, retries=MAX, max.in.flight.requests=5
```

> 幂等性原理：`<PID, Partition, SeqNumber>` 三元组去重。

#### 事务（跨分区 Exactly Once）

```java
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "my-transaction-id");
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

### 2.5 数据有序性 ⭐

```
分区内有序保证:
1. 开启幂等性（默认）: max.in.flight.requests ≤ 5 时自动保证
2. 未开启幂等性: max.in.flight.requests = 1（但吞吐量低）

全局有序: 只设置 1 个分区（不推荐，性能差）
```

---

## 第3章 Kafka Broker 调优 🔥

### 3.1 核心参数

| 参数 | 建议值 | 说明 |
|:---|:---|:---|
| `num.io.threads` | **8~16** | 处理磁盘 IO 的线程数 |
| `num.network.threads` | **6~9** | 处理网络请求的线程数 |
| `num.replica.fetchers` | **4** | 副本同步线程数 |
| `log.retention.hours` | **168（7天）** | 日志保留时间 |
| `log.segment.bytes` | **1GB** | 单个 Segment 大小 |
| `log.cleanup.policy` | **delete** | 清理策略（delete/compact） |

### 3.2 服役新节点 / 退役旧节点

```bash
# 1. 生成迁移计划
kafka-reassign-partitions.sh --bootstrap-server hadoop102:9092 \
    --topics-to-move-json-file topics.json \
    --broker-list "0,1,2,3" \
    --generate

# 2. 执行迁移
kafka-reassign-partitions.sh --bootstrap-server hadoop102:9092 \
    --reassignment-json-file plan.json \
    --execute

# 3. 验证迁移
kafka-reassign-partitions.sh --bootstrap-server hadoop102:9092 \
    --reassignment-json-file plan.json \
    --verify
```

### 3.3 Leader Partition 负载均衡

```properties
# 自动均衡
auto.leader.rebalance.enable = true
# 不均衡比例超过10%触发
leader.imbalance.per.broker.percentage = 10
# 检查间隔
leader.imbalance.check.interval.seconds = 300
```

---

## 第4章 Kafka 消费者调优 🔥

### 4.1 核心参数

| 参数 | 建议 | 说明 |
|:---|:---|:---|
| `fetch.max.bytes` | **52428800 (50MB)** | 一次拉取最大数据量 |
| `max.poll.records` | **500** | 一次 poll 最大记录数 |
| `max.poll.interval.ms` | **300000 (5min)** | 两次 poll 最大间隔 |
| `session.timeout.ms` | **45000** | 会话超时时间 |
| `heartbeat.interval.ms` | **3000** | 心跳间隔 |

### 4.2 提高消费吞吐量

```java
// 1. 增加每次拉取数据量
props.put(ConsumerConfig.FETCH_MAX_BYTES_CONFIG, 52428800);  // 50MB

// 2. 增加单次 poll 记录数
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);

// 3. 增加消费者数量（= 分区数）
// 消费者数 = 分区数 时最优

// 4. 下游处理用异步 + 多线程
```

### 4.3 消费者再平衡

```
触发条件:
1. 消费者加入或离开 Consumer Group
2. 订阅 Topic 的分区数变化
3. 消费者超时（超过 max.poll.interval.ms 没有 poll）

影响: 再平衡期间消费者停止消费！

减少再平衡:
1. 增大 session.timeout.ms (45s)
2. 增大 max.poll.interval.ms (5min)
3. 减少单次处理时间
```

---

## 第5章 Kafka 总体调优 🔥🔥

### 5.1 数据精准一次（端到端 Exactly Once）

```
Producer → Kafka → Consumer 端到端精准一次:

1. Producer: 开启幂等 + 事务
2. Broker: acks=-1, min.insync.replicas=2
3. Consumer: 手动提交 Offset + 处理幂等（或事务写入）
```

### 5.2 合理设置分区数 🔥

```
分区数设置原则:
1. 生产端: 生产速度 / 单分区写入速度
2. 消费端: 消费速度 / 单消费者消费速度
3. 取两者最大值

示例: 峰值 20MB/s, 单分区写入 10MB/s, 单消费者 5MB/s
  生产端: 20/10 = 2
  消费端: 20/5 = 4
  → 分区数 = max(2, 4) = 4 个分区
```

### 5.3 单条日志大于 1MB 的处理

```properties
# Broker
message.max.bytes = 10485760         # 10MB
replica.fetch.max.bytes = 10485760

# Producer
max.request.size = 10485760

# Consumer
max.partition.fetch.bytes = 10485760
fetch.max.bytes = 52428800
```

### 5.4 集群压力测试

```bash
# 生产者压测
kafka-producer-perf-test.sh \
    --topic test-perf \
    --num-records 1000000 \
    --record-size 1024 \
    --throughput -1 \
    --producer-props bootstrap.servers=hadoop102:9092 \
    batch.size=32768 linger.ms=5

# 消费者压测
kafka-consumer-perf-test.sh \
    --bootstrap-server hadoop102:9092 \
    --topic test-perf \
    --messages 1000000
```

---

## 🔥 面试高频题

### Q1：Kafka 如何提高吞吐量？
> Producer：增大 batch.size、linger.ms、buffer.memory，开启压缩。Broker：增加分区数，优化磁盘 IO 线程。Consumer：增加消费者数量（=分区数），增大 fetch 参数，下游异步处理。

### Q2：如何保证 Kafka 数据不丢失？
> Producer：acks=-1 + retries > 0。Broker：min.insync.replicas=2 + unclean.leader.election.enable=false。Consumer：手动提交 Offset（处理完再提交）。

### Q3：Kafka 分区数如何确定？
> 取生产端和消费端需要的分区数最大值。生产端 = 目标吞吐量/单分区写入速度，消费端 = 目标吞吐量/单消费者消费速度。建议先用压测确定单分区/单消费者的吞吐量上限。
