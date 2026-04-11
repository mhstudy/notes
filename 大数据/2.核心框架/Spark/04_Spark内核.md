# Spark 内核

> 🔗 **官方文档**：https://spark.apache.org/docs/latest/
> 📌 **学习版本**：Spark 3.x

---

## 第1章 Spark 内核概述 ⭐

### 1.1 核心组件回顾

| 组件 | 说明 |
|:---|:---|
| **Driver** | 运行 main 方法，创建 SparkContext，生成 DAG，划分 Stage，调度 Task |
| **Executor** | 在 Worker 节点运行，执行 Task，缓存 RDD 数据 |
| **Cluster Manager** | 资源管理（Standalone / YARN / Mesos / K8s） |

### 1.2 Spark 通用运行流程 🔥🔥

```
1. Driver 创建 SparkContext
2. SparkContext 向 Cluster Manager 注册并申请资源
3. Cluster Manager 在 Worker 上启动 Executor
4. Executor 向 Driver 注册
5. SparkContext 构建 DAG → 划分 Stage → 生成 TaskSet
6. TaskScheduler 将 Task 分配到 Executor 执行
7. Task 执行完毕，释放资源
```

---

## 第2章 Spark 部署模式 🔥

### 2.1 YARN 模式 🔥🔥

#### YARN Cluster 模式（生产）

```
Client 提交 → RM 分配 Container → 启动 AM(即Driver)
AM 向 RM 申请资源 → 在 NM 上启动 Executor → 执行 Task
```

> 💡 Driver 运行在 AM 中，**Client 提交后可断开**。

#### YARN Client 模式（调试）

```
Client 启动 Driver → Driver 向 RM 申请资源
RM 在 NM 上启动 AM(不含Driver) → AM 向 RM 申请 Executor
Executor 向 Driver 注册 → 执行 Task
```

> 💡 Driver 运行在 Client 端，**可以看到实时日志**，适合调试。

### 2.2 Cluster vs Client 对比 🔥

| | Cluster 模式 | Client 模式 |
|:---|:---|:---|
| Driver 位置 | AM 中（集群内） | 客户端（集群外） |
| 日志查看 | YARN Web UI | 客户端控制台 |
| 网络依赖 | 低（Driver 在集群内） | 高（Task 结果回传 Client） |
| 适用场景 | **生产环境** | 调试/测试 |

```bash
# Cluster 模式提交
spark-submit --master yarn --deploy-mode cluster \
    --class com.atguigu.spark.WordCount spark-demo.jar

# Client 模式提交
spark-submit --master yarn --deploy-mode client \
    --class com.atguigu.spark.WordCount spark-demo.jar
```

---

## 第3章 Spark 通讯架构 📝

```
Spark 通讯基于 Netty（RPC 框架）:

Driver                        Executor
  │                              │
  │  HeartbeatReceiver          │
  │←──────── 心跳 ──────────────│ (每 10s)
  │                              │
  │  CoarseGrainedScheduler     │
  │──────── LaunchTask ────────→│
  │                              │
  │←──────── StatusUpdate ──────│ (Task完成/失败)
  │                              │
```

---

## 第4章 Spark 任务调度 🔥🔥

### 4.1 调度概述

```
Application → 多个 Job
Job → 多个 Stage（遇到宽依赖划分）
Stage → 多个 Task（一个分区一个 Task）
```

### 4.2 DAGScheduler（Stage 级调度）🔥🔥

```
RDD 依赖关系 → DAG 有向无环图 → 按宽依赖划分 Stage

示例:
textFile → flatMap → map → reduceByKey → collect
   ↓         ↓        ↓        ↓           ↓
  RDD1     RDD2     RDD3    Shuffle!      结果
  
Stage 划分:
  ShuffleMapStage (textFile → flatMap → map)
        ↓ Shuffle
  ResultStage (reduceByKey → collect)
```

**Stage 划分规则**：
1. 从最后一个 RDD 往前回溯
2. 遇到**宽依赖**（Shuffle）就切分一个新 Stage
3. 遇到**窄依赖**则合并到同一 Stage

### 4.3 TaskScheduler（Task 级调度）🔥

#### 调度策略

| 策略 | 说明 |
|:---|:---|
| **FIFO（默认）** | 先进先出，按 Job 提交顺序执行 |
| **FAIR** | 公平调度，资源在多个 Job 间共享 |

```scala
// 设置公平调度
conf.set("spark.scheduler.mode", "FAIR")
```

#### 本地化调度 🔥

| 本地化级别 | 说明 | 性能 |
|:---|:---|:---|
| **PROCESS_LOCAL** | Task 和数据在同一 Executor | 最快 |
| **NODE_LOCAL** | Task 和数据在同一节点不同 Executor | 快 |
| **NO_PREF** | 数据无所谓位置 | 中 |
| **RACK_LOCAL** | Task 和数据在同一机架 | 慢 |
| **ANY** | 跨机架 | 最慢 |

> 💡 TaskScheduler 优先按最高本地化级别分配，等待超时后降级。

```scala
// 本地化等待时间
spark.locality.wait = 3s            // 总等待时间
spark.locality.wait.process = 3s    // PROCESS_LOCAL 等待
spark.locality.wait.node = 3s       // NODE_LOCAL 等待
spark.locality.wait.rack = 3s       // RACK_LOCAL 等待
```

#### 失败重试与黑名单

```scala
spark.task.maxFailures = 4      // Task 最大失败重试次数
spark.blacklist.enabled = true  // 启用黑名单（多次失败的节点加入黑名单）
```

---

## 第5章 Spark Shuffle 解析 🔥🔥🔥

### 5.1 Shuffle 核心要点

Shuffle 是 Spark 中最**消耗性能**的操作：
- 涉及**磁盘 IO**（溢写、读取）
- 涉及**网络 IO**（跨节点数据传输）
- 涉及**序列化/反序列化**

### 5.2 HashShuffle（已废弃）📝

```
未优化版:
  每个 MapTask 为每个 ReduceTask 生成一个文件
  → M 个 MapTask × R 个 ReduceTask = M × R 个文件（文件爆炸！）

优化版(Consolidate):
  同一 Core 上的 MapTask 共享文件
  → Core 数 × R 个文件
```

### 5.3 SortShuffle（当前默认）🔥🔥

#### 普通 SortShuffle

```
MapTask 输出:
1. 数据写入内存缓冲区（5MB 默认）
2. 达到阈值 → 溢写到磁盘（排序 + 分区）
3. 多个溢写文件 → 归并排序 → 1 个数据文件 + 1 个索引文件

每个 MapTask 最终只产生 2 个文件:
  - 数据文件（按分区排序）
  - 索引文件（记录每个分区在数据文件中的位置）
```

#### Bypass SortShuffle 🔥

当满足以下条件时，**跳过排序**：
1. Shuffle Map Task 数 ≤ `spark.shuffle.sort.bypassMergeThreshold`（默认 200）
2. 不需要 Map 端预聚合

```
Bypass 流程:
  不排序，直接按分区写文件 → 合并为 1 个文件 + 索引
  → 减少了排序开销
```

---

## 第6章 Spark 内存管理 🔥🔥

### 6.1 堆内和堆外内存

```
Executor 内存布局 (统一内存管理 - Spark 1.6+):

堆内内存 (spark.executor.memory):
┌──────────────────────────────────┐
│  预留内存 (300MB 固定)            │
├──────────────────────────────────┤
│  统一内存 (默认 60%)              │
│  ┌─────────────┬────────────┐   │
│  │ 存储内存    ↔  执行内存    │   │  ← 可以互相借用
│  │ (缓存RDD)     (Shuffle等) │   │
│  └─────────────┴────────────┘   │
├──────────────────────────────────┤
│  用户内存 (默认 40%)              │  ← 用户定义的数据结构
└──────────────────────────────────┘

堆外内存 (spark.memory.offHeap.size):
┌──────────────────────────────────┐
│  存储内存    ↔    执行内存        │  ← 同样可互相借用
└──────────────────────────────────┘
```

### 6.2 统一内存管理关键参数

| 参数 | 默认值 | 说明 |
|:---|:---|:---|
| `spark.memory.fraction` | **0.6** | 统一内存占比（堆 - 300MB） |
| `spark.memory.storageFraction` | **0.5** | 存储内存占统一内存比例 |
| `spark.memory.offHeap.enabled` | false | 是否启用堆外内存 |
| `spark.memory.offHeap.size` | 0 | 堆外内存大小 |

### 6.3 存储内存与执行内存的动态占用 🔥

```
规则:
1. 双方互相借用对方空闲内存
2. 存储内存被执行内存借用后，存储数据需淘汰（LRU）
3. 执行内存被存储内存借用后，执行不能强制淘汰（等待释放）
```

> 💡 执行内存优先级更高（不能被强制收回），因为 Shuffle 中途停止代价太大。

---

## 🔥 面试高频题

### Q1：Spark 的 Stage 是如何划分的？
> 从最后一个 RDD 向前回溯 DAG 图，遇到宽依赖（Shuffle）就划分为一个新 Stage。Stage 分为 ShuffleMapStage（产生 Shuffle 数据）和 ResultStage（产生最终结果）。

### Q2：Spark 的 Shuffle 过程？
> 当前默认使用 SortShuffle：Map 端数据写入缓冲区，溢写时按分区排序写入磁盘，最终合并为 1 个数据文件 + 1 个索引文件。Reduce 端通过索引文件拉取对应分区数据。如果分区数 ≤ 200 且无预聚合，使用 Bypass SortShuffle（跳过排序）。

### Q3：Spark 的内存管理？
> 统一内存管理（1.6+）：Executor 内存分为预留（300MB）+ 统一内存（60%）+ 用户内存（40%）。统一内存中存储内存和执行内存各占 50%，可动态借用。执行内存优先级更高，被借用后不会被强制收回。

### Q4：YARN Cluster 和 Client 模式的区别？
> Cluster 模式 Driver 运行在 AM 中（集群内），Client 提交后可断开；Client 模式 Driver 运行在提交机器上，可实时看日志但网络开销大。生产用 Cluster，调试用 Client。
