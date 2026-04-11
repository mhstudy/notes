# Kafka 源码解析

> 🔗 **源码地址**：https://github.com/apache/kafka
> 📌 **学习版本**：Kafka 3.3.x

---

## 第1章 源码环境准备 📝

```bash
# 下载源码
git clone https://github.com/apache/kafka.git
cd kafka
git checkout 3.3.1

# 使用 Gradle 构建
./gradlew idea   # 生成 IDEA 项目文件
```

---

## 第2章 生产者源码 🔥🔥

### 2.1 Producer 初始化

```
KafkaProducer 构造函数:
  ├→ 解析配置参数
  ├→ 创建 Serializer（Key/Value 序列化器）
  ├→ 创建 Partitioner（分区器）
  ├→ 创建 RecordAccumulator（消息累加器/缓冲区，默认32MB）
  ├→ 创建 NetworkClient（网络客户端）
  └→ 创建并启动 Sender 线程（守护线程，负责实际网络发送）
```

```java
// 核心对象关系:
// KafkaProducer(主线程) → RecordAccumulator(缓冲区) → Sender(IO线程) → NetworkClient → Broker
```

### 2.2 发送数据到缓冲区 🔥

```
producer.send() 流程:
1. 拦截器 ProducerInterceptor.onSend()
2. 序列化 Key 和 Value
3. 分区器计算目标分区
4. 消息大小校验（不超过 max.request.size 和 buffer.memory）
5. 追加到 RecordAccumulator 的对应分区队列 (Deque<ProducerBatch>)
6. 如果 batch 满了或新建了 batch → 唤醒 Sender 线程
```

#### 分区选择策略 🔥

```java
// Kafka 3.x 默认分区器: DefaultPartitioner → 粘性分区
// 1. 指定 partition → 直接使用
// 2. 指定 key → hash(key) % numPartitions
// 3. 都没指定 → 粘性分区（同一 batch 内发往同一分区，batch 满后换分区）
```

#### 内存池（BufferPool）

```
RecordAccumulator 使用 BufferPool 管理内存:
- 总大小: buffer.memory (默认 32MB)
- 每个 ProducerBatch: batch.size (默认 16KB)
- 申请内存时，优先从空闲池获取，不够则分配新内存
- batch 发送完毕后，内存归还到空闲池
- 内存不足时，producer.send() 阻塞等待（最多 max.block.ms）
```

### 2.3 Sender 线程发送数据 🔥

```
Sender.run() 循环:
1. 从 RecordAccumulator 获取准备好的 ProducerBatch
   - batch 满了
   - 等待时间 ≥ linger.ms
   - 缓冲区内存不足
   - close() 被调用
2. 将 batch 按 Broker 分组
3. 构建 ProduceRequest
4. 通过 NetworkClient 发送到对应 Broker
5. 处理 Broker 的 Response
   - 成功: 从 RecordAccumulator 移除 batch，执行回调
   - 失败: 重试（retries 未耗尽）或执行失败回调
```

---

## 第3章 消费者源码 🔥

### 3.1 Consumer 初始化

```
KafkaConsumer 构造函数:
  ├→ 解析配置参数
  ├→ 创建 Deserializer（反序列化器）
  ├→ 创建 ConsumerCoordinator（协调器，负责再平衡）
  ├→ 创建 Fetcher（数据拉取器）
  └→ 创建 NetworkClient
```

### 3.2 消费者订阅主题

```java
// consumer.subscribe() 并不立即消费
// 只是记录订阅的 Topic，等 poll() 时才真正开始

consumer.subscribe(Arrays.asList("topic1", "topic2"));
// 内部: subscriptions.subscribe(topics, listener)
```

### 3.3 消费者拉取数据 🔥

```
consumer.poll() 流程:
1. 检查是否需要再平衡（ConsumerCoordinator）
   - 首次 poll: 加入消费者组 → JoinGroup → SyncGroup → 获得分区分配
   - 非首次: 检查 heartbeat 是否正常
2. 更新 Offset 位置（从哪里开始消费）
   - auto.offset.reset: latest / earliest / none
3. Fetcher 向 Broker 发送 FetchRequest
4. 等待响应，解析数据
5. 执行 ConsumerInterceptor.onConsume()
6. 返回 ConsumerRecords
```

### 3.4 消费者组再平衡 🔥🔥

```
再平衡流程:
1. Consumer → Coordinator: JoinGroupRequest（携带支持的分配策略）
2. Coordinator 选出 Leader Consumer（第一个加入的）
3. Coordinator → All Consumers: JoinGroupResponse
   - Leader 收到所有成员信息 + 决定分区分配方案
   - Follower 收到空响应
4. Leader Consumer: 执行分区分配（Range/RoundRobin/Sticky/CooperativeSticky）
5. All Consumers → Coordinator: SyncGroupRequest
   - Leader 上传分配结果
   - Follower 发送空请求
6. Coordinator → All Consumers: SyncGroupResponse（含分配结果）
7. 各 Consumer 开始按分配的分区消费
```

### 3.5 Offset 提交 ⭐

```java
// 自动提交（默认）
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, 5000);
// 风险: 处理失败但 Offset 已提交 → 丢数据

// 手动同步提交
consumer.commitSync();  // 阻塞到提交成功

// 手动异步提交
consumer.commitAsync((offsets, exception) -> {
    if (exception != null) {
        log.error("Commit failed: " + offsets, exception);
    }
});
```

---

## 第4章 Broker 源码 📝

### 4.1 Broker 启动流程

```
KafkaServer.startup():
  ├→ 初始化 ZK 连接（或 KRaft Controller）
  ├→ 创建 LogManager（日志管理，负责 Topic 分区数据）
  ├→ 创建 ReplicaManager（副本管理）
  ├→ 创建 GroupCoordinator（消费者组协调器）
  ├→ 创建 KafkaApis（请求处理器）
  ├→ 创建 SocketServer（网络层）
  │   ├→ Acceptor 线程：接受 TCP 连接
  │   └→ Processor 线程池：处理网络 IO
  └→ 启动 KafkaRequestHandlerPool（请求处理线程池）
```

### 4.2 请求处理模型（Reactor 模式）🔥

```
                  ┌── Processor 1 ──┐
Client ──→ Acceptor ──→ Processor 2 ──→ RequestQueue ──→ Handler线程池
                  └── Processor N ──┘                      │
                                                           ▼
                                                      KafkaApis
                                                     (处理各种请求)
                                                           │
                                                           ▼
                                                      ResponseQueue
                                                           │
                                                           ▼
                                                    Processor → Client
```

| 线程 | 数量配置 | 职责 |
|:---|:---|:---|
| Acceptor | 1 | 接受新的 TCP 连接 |
| Processor | `num.network.threads` (默认3) | 处理网络 IO |
| Handler | `num.io.threads` (默认8) | 处理实际业务逻辑 |

---

## 🔥 面试高频题

### Q1：Kafka Producer 的发送流程？
> 主线程：拦截器 → 序列化 → 分区器 → 写入 RecordAccumulator 缓冲区。Sender 线程：从缓冲区取出满足条件的 batch → 构建请求 → NetworkClient 发送到 Broker → 处理响应（成功/重试/失败回调）。

### Q2：Kafka 消费者组再平衡流程？
> Consumer 发送 JoinGroupRequest → Coordinator 选 Leader Consumer → Leader 执行分区分配策略 → SyncGroupRequest 上传/获取分配结果 → 各 Consumer 按分配消费。再平衡期间消费暂停。

### Q3：Kafka Broker 的网络模型？
> Reactor 模型：1 个 Acceptor 线程接受连接，N 个 Processor 线程处理网络 IO，M 个 Handler 线程处理业务逻辑。请求通过 RequestQueue 传递给 Handler，响应通过 ResponseQueue 返回给 Processor。
