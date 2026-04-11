# Hadoop 源码解析

> 🔗 **源码地址**：https://github.com/apache/hadoop
> 📌 **学习版本**：Hadoop 3.3.x

---

## 第0章 RPC 通信原理 📝

Hadoop 内部通信基于 **RPC（Remote Procedure Call）** 框架实现。

```
Client                          Server
  │                               │
  │  1. 创建代理对象(Proxy)         │
  │───────────────────────────→   │
  │  2. 序列化请求 (Protobuf)      │
  │───────────────────────────→   │
  │                               │ 3. 反序列化，调用本地方法
  │  4. 返回序列化结果              │
  │←───────────────────────────   │
  │  5. 反序列化结果               │
  │                               │
```

> Hadoop RPC 底层用 **Protobuf** 序列化，通过 **Socket** 通信。

---

## 第1章 NameNode 启动源码 🔥

### 1.1 核心启动流程

```
NameNode.main()
    └→ createNameNode()
        └→ new NameNode(conf)
            ├→ 启动 HTTP 服务 (9870端口)
            ├→ 加载 Fsimage + Edits（元数据恢复）
            ├→ 初始化 RPC 服务端 (8020端口)
            ├→ 启动资源检查（磁盘空间）
            └→ 进入安全模式，等待 DN 心跳
```

### 1.2 加载镜像文件和编辑日志 🔥

```
启动时元数据恢复流程:
1. 加载 Fsimage（内存快照）
2. 回放 Edits（编辑日志）中的操作
3. 合并后得到最新的内存元数据
4. 保存新的 Fsimage（Checkpoint）
```

> 💡 **面试**：2NN 的作用？  
> 答：2NN 定期合并 Fsimage 和 Edits，减少 NN 启动时间。合并条件：Edits 数量达 100 万或距上次合并超过 1 小时。

### 1.3 NN 安全模式判断

```java
// 安全模式退出条件
// 1. Block 上报比例达到阈值（默认 0.999）
// 2. DataNode 数量达到最小要求
// 3. 满足条件后等待 30 秒
dfs.namenode.safemode.threshold-pct = 0.999f
dfs.namenode.safemode.min.datanodes = 0
dfs.namenode.safemode.extension = 30000  // ms
```

---

## 第2章 DataNode 启动源码 ⭐

### 2.1 核心启动流程

```
DataNode.main()
    └→ createDataNode()
        └→ new DataNode(conf)
            ├→ 初始化 DataXceiverServer（数据传输服务）
            ├→ 启动 HTTP 服务
            ├→ 初始化 RPC 客户端
            ├→ 向 NN 注册
            └→ 开始定期发送心跳
```

### 2.2 DN 向 NN 心跳机制 🔥

```
心跳内容:
1. DN 存储容量和使用情况
2. DN 上的 Block 列表（块报告）
3. DN 的健康状态

心跳间隔: dfs.heartbeat.interval = 3 秒
块报告间隔: dfs.blockreport.intervalMsec = 6 小时

超时判断: timeout = 2 * heartbeat.recheck-interval + 10 * heartbeat.interval
         默认 = 2 * 5min + 10 * 3s = 10min30s
```

> ⚠️ DN 超过 10 分 30 秒无心跳 → NN 认为该 DN 已死，不再分配新的读写任务。

---

## 第3章 HDFS 上传源码解析 🔥🔥

### 3.1 create 创建过程

```
Client.create(path)
  └→ DFSClient.create()
      ├→ RPC 调用 NN.create() -- 检查权限、创建文件元数据
      └→ 返回 DFSOutputStream
          └→ 启动 DataStreamer 线程

DataStreamer 职责:
1. 向 NN 申请 Block 存储位置
2. 建立到 DN 的 Pipeline
3. 将数据包发送到 Pipeline
```

### 3.2 write 上传过程 🔥

```
Client 写数据流程:
1. 数据写入 → DFSOutputStream → 切分为 Packet（64KB）
2. Packet 放入 dataQueue（数据队列）
3. DataStreamer 从 dataQueue 取 Packet
4. 向 NN 申请 Block 位置 → 返回 DN 列表
5. 建立 Pipeline (DN1 → DN2 → DN3)
6. Packet 发送 → 移入 ackQueue（确认队列）
7. DN3→DN2→DN1 逐级应答 → 从 ackQueue 移除
```

```
数据包传输:
┌────────┐    Pipeline    ┌─────┐    ┌─────┐    ┌─────┐
│ Client │ ──Packet──→   │ DN1 │ ──→│ DN2 │ ──→│ DN3 │
│        │ ←──ACK────    │     │ ←──│     │ ←──│     │
└────────┘               └─────┘    └─────┘    └─────┘
```

### 3.3 机架感知（副本存储位置）🔥

```
3副本存储策略:
- 第1个副本: Client 所在节点（Client 不在集群则随机选一个不太忙的节点）
- 第2个副本: 与第1个副本不同机架的随机节点
- 第3个副本: 与第2个副本同机架的不同节点
```

> 💡 设计目的：兼顾**可靠性**（跨机架容灾）和**性能**（同机架带宽大）。

---

## 第4章 HDFS 下载源码解析 ⭐

```
Client.open(path)
  └→ DFSClient.open()
      ├→ RPC 调用 NN.getBlockLocations() -- 获取 Block 位置列表
      └→ 返回 DFSInputStream

Client.read()
  └→ DFSInputStream.read()
      ├→ 选择最近的 DN（网络拓扑距离最短）
      ├→ 建立 Socket 连接读取数据
      ├→ 读完一个 Block，关闭连接
      └→ 连接下一个 Block 的最近 DN
```

---

## 第5章 YARN 源码解析 ⭐

### 5.1 Job 提交流程

```
Client                    RM                      NM
  │ 1.submitJob            │                       │
  │──────────────────→     │                       │
  │ 2.返回 ApplicationId   │                       │
  │←──────────────────     │                       │
  │ 3.提交资源(jar/xml/split) → HDFS              │
  │ 4.submitApplication    │                       │
  │──────────────────→     │                       │
  │                        │ 5.启动Container       │
  │                        │──────────────────→    │
  │                        │  启动MRAppMaster      │
  │                        │                    6.初始化Task
  │                        │                    7.领取Task
  │                        │←──────────────────    │
  │                        │ 8.分配Container       │
  │                        │──────────────────→    │
  │                        │                    9.启动YarnChild
  │                        │                   10.执行MapTask/ReduceTask
```

### 5.2 调度器核心逻辑 📝

```java
// 容量调度器 CapacityScheduler
// 调度策略: 优先选择资源使用率最低的队列
// 队列内部: 按提交时间排序 (FIFO)
// 支持: 队列弹性（max-capacity > capacity 时可借用资源）

// 公平调度器 FairScheduler  
// 调度策略: 优先选择资源缺额最大的队列
// 队列内部: 按公平排序 (资源使用少的优先)
// 支持: 抢占（资源不归还时可强制回收）
```

---

## 第6章 MapReduce 源码解析 📝

### 6.1 Job 提交切片流程

```java
// 切片核心逻辑 FileInputFormat.getSplits()
long splitSize = computeSplitSize(blockSize, minSize, maxSize);
// splitSize = Math.max(minSize, Math.min(maxSize, blockSize))

// 遍历每个文件
for each file:
    long bytesRemaining = fileLength;
    while (bytesRemaining / splitSize > 1.1) {
        splits.add(new FileSplit(path, start, splitSize));
        start += splitSize;
        bytesRemaining -= splitSize;
    }
    // 剩余部分作为最后一个切片
    splits.add(new FileSplit(path, start, bytesRemaining));
```

### 6.2 MapTask 工作机制

```
1. Read 阶段:   InputFormat.RecordReader 读取切片数据
2. Map 阶段:    用户 Mapper.map() 逻辑处理
3. Collect 阶段: 输出 KV → 环形缓冲区(kvbuffer, 默认100MB)
4. Spill 阶段:   80% → 溢写（快排、分区、可选Combiner）
5. Merge 阶段:  多个溢写文件归并排序
```

### 6.3 ReduceTask 工作机制

```
1. Copy 阶段:   Fetcher 线程从各 MapTask 拉取对应分区数据
2. Sort 阶段:   归并排序所有拉取的数据（分组排序）
3. Reduce 阶段: 调用用户 Reducer.reduce() 逻辑处理
```

---

## 🔥 面试高频题

### Q1：NameNode 启动时做了什么？
> 加载 Fsimage 到内存，回放 Edits 日志恢复最新元数据，启动 RPC 和 HTTP 服务，进入安全模式等待 DN 心跳上报 Block，Block 上报率 ≥ 99.9% 后退出安全模式。

### Q2：HDFS 的副本存储策略（机架感知）？
> 第1个副本在Client所在节点，第2个副本在不同机架的随机节点，第3个副本在第2个副本同机架的不同节点。兼顾可靠性和性能。

### Q3：DataNode 掉线时限如何计算？
> timeout = 2 × recheck-interval（默认5分钟） + 10 × heartbeat.interval（默认3秒） = **10分30秒**。超时后 NN 将该 DN 标记为死亡。
