# Hadoop HA 高可用

> 🔗 **官方文档**：https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HDFSHighAvailabilityWithQJM.html
> 📌 **学习版本**：Hadoop 3.3.x

---

## 第1章 HA 概述 🔥

### 1.1 为什么需要 HA

单 NameNode 存在**单点故障（SPOF）**问题：
- NN 宕机 → 整个集群不可用
- NN 维护 → 集群停服

### 1.2 HA 架构 🔥🔥

```
┌──────────────────────────────────────────────────┐
│                   ZooKeeper 集群                   │
│              (Leader选举 + 健康检测)                │
└─────────┬──────────────────┬─────────────────────┘
          │                  │
    ┌─────┴─────┐      ┌─────┴─────┐
    │  ZKFC     │      │  ZKFC     │     ← ZooKeeper Failover Controller
    │           │      │           │
    │ NameNode  │      │ NameNode  │
    │ (Active)  │      │ (Standby) │
    └─────┬─────┘      └─────┬─────┘
          │    共享编辑日志    │
          │  ┌────────────┐  │
          └──│ JournalNode │──┘
             │   集群(3台)  │
             └────────────┘
                   │
          ┌────────┼────────┐
       DataNode  DataNode  DataNode
       (同时向两个 NN 汇报 Block)
```

---

## 第2章 HDFS HA 工作机制 🔥🔥

### 2.1 核心组件

| 组件 | 说明 |
|:---|:---|
| **Active NameNode** | 对外提供读写服务 |
| **Standby NameNode** | 同步 Active 的元数据，随时准备接管 |
| **JournalNode（JN）** | 共享存储系统，存放 Edits 编辑日志 |
| **ZKFC** | 监控 NN 健康状态，通过 ZK 实现自动故障转移 |
| **ZooKeeper** | 选举 Active NN，存储状态信息 |

### 2.2 元数据同步机制

```
Active NN 写操作:
1. Active NN 将编辑日志同时写入 JournalNode 集群（多数派写入成功即可）
2. Standby NN 定期从 JournalNode 读取新的编辑日志
3. Standby NN 将编辑日志应用到内存中的文件系统
4. DataNode 同时向两个 NN 发送 Block 报告
```

> 💡 JournalNode 使用 **Paxos** 协议保证数据一致性，至少 3 台（2N+1）。

### 2.3 故障转移流程 🔥🔥

```
正常情况:
  ZKFC-1 ──健康──→ Active NN-1  ← 对外服务
  ZKFC-2 ──健康──→ Standby NN-2 ← 同步日志

故障转移:
  1. ZKFC-1 检测到 NN-1 心跳超时（HealthMonitor）
  2. ZKFC-1 在 ZK 上释放 Active 锁
  3. ZKFC-2 在 ZK 上抢到 Active 锁
  4. ZKFC-2 通知 NN-2 切换为 Active
  5. ZKFC-2 对 NN-1 执行 fencing（隔离，防止脑裂）
  6. NN-2 成为新的 Active，对外提供服务
```

### 2.4 防止脑裂（Fencing）🔥

防止两个 NN 同时为 Active 的机制：

| Fencing 方式 | 说明 |
|:---|:---|
| **sshfence** | SSH 登录到旧 Active，杀死 NN 进程 |
| **shell** | 执行自定义 Shell 脚本 |

```xml
<!-- hdfs-site.xml -->
<property>
    <name>dfs.ha.fencing.methods</name>
    <value>sshfence</value>
</property>
<property>
    <name>dfs.ha.fencing.ssh.private-key-files</name>
    <value>/home/hadoop/.ssh/id_rsa</value>
</property>
```

---

## 第3章 HDFS HA 配置 🔥

### 3.1 集群规划

| | hadoop102 | hadoop103 | hadoop104 |
|:---|:---|:---|:---|
| NameNode | ✅ | ✅ | |
| JournalNode | ✅ | ✅ | ✅ |
| DataNode | ✅ | ✅ | ✅ |
| ZooKeeper | ✅ | ✅ | ✅ |
| ZKFC | ✅ | ✅ | |

### 3.2 核心配置

```xml
<!-- hdfs-site.xml -->
<!-- 命名空间 -->
<property>
    <name>dfs.nameservices</name>
    <value>mycluster</value>
</property>

<!-- 命名空间下的 NN -->
<property>
    <name>dfs.ha.namenodes.mycluster</name>
    <value>nn1,nn2</value>
</property>

<!-- NN RPC 地址 -->
<property>
    <name>dfs.namenode.rpc-address.mycluster.nn1</name>
    <value>hadoop102:8020</value>
</property>
<property>
    <name>dfs.namenode.rpc-address.mycluster.nn2</name>
    <value>hadoop103:8020</value>
</property>

<!-- NN Web 地址 -->
<property>
    <name>dfs.namenode.http-address.mycluster.nn1</name>
    <value>hadoop102:9870</value>
</property>
<property>
    <name>dfs.namenode.http-address.mycluster.nn2</name>
    <value>hadoop103:9870</value>
</property>

<!-- JournalNode 地址 -->
<property>
    <name>dfs.namenode.shared.edits.dir</name>
    <value>qjournal://hadoop102:8485;hadoop103:8485;hadoop104:8485/mycluster</value>
</property>

<!-- 故障转移代理 -->
<property>
    <name>dfs.client.failover.proxy.provider.mycluster</name>
    <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
</property>

<!-- Fencing -->
<property>
    <name>dfs.ha.fencing.methods</name>
    <value>sshfence</value>
</property>

<!-- 自动故障转移 -->
<property>
    <name>dfs.ha.automatic-failover.enabled</name>
    <value>true</value>
</property>
```

```xml
<!-- core-site.xml -->
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://mycluster</value>
</property>
<property>
    <name>ha.zookeeper.quorum</name>
    <value>hadoop102:2181,hadoop103:2181,hadoop104:2181</value>
</property>
```

### 3.3 启动命令

```bash
# 1. 启动 JournalNode
hdfs --daemon start journalnode  # 三台都执行

# 2. 格式化 NN（仅首次）
hdfs namenode -format
# 启动 nn1
hdfs --daemon start namenode

# 3. 同步 nn1 元数据到 nn2
hdfs namenode -bootstrapStandby  # 在 hadoop103 执行

# 4. 格式化 ZKFC
hdfs zkfc -formatZK

# 5. 启动 HDFS（会自动启动 ZKFC）
start-dfs.sh
```

---

## 第4章 YARN HA ⭐

### 4.1 YARN HA 架构

```
┌───────────────────────────────────────┐
│           ZooKeeper 集群              │
└─────────┬──────────────┬──────────────┘
          │              │
   ResourceManager    ResourceManager
    (Active)           (Standby)
          │              │
     ┌────┼────┬────┐    │
     NM   NM   NM   NM  │
```

### 4.2 YARN HA 配置

```xml
<!-- yarn-site.xml -->
<property>
    <name>yarn.resourcemanager.ha.enabled</name>
    <value>true</value>
</property>
<property>
    <name>yarn.resourcemanager.cluster-id</name>
    <value>cluster-yarn1</value>
</property>
<property>
    <name>yarn.resourcemanager.ha.rm-ids</name>
    <value>rm1,rm2</value>
</property>
<property>
    <name>yarn.resourcemanager.hostname.rm1</name>
    <value>hadoop102</value>
</property>
<property>
    <name>yarn.resourcemanager.hostname.rm2</name>
    <value>hadoop103</value>
</property>
<property>
    <name>yarn.resourcemanager.webapp.address.rm1</name>
    <value>hadoop102:8088</value>
</property>
<property>
    <name>yarn.resourcemanager.webapp.address.rm2</name>
    <value>hadoop103:8088</value>
</property>
<property>
    <name>yarn.resourcemanager.zk-address</name>
    <value>hadoop102:2181,hadoop103:2181,hadoop104:2181</value>
</property>
<!-- 启用自动故障转移 -->
<property>
    <name>yarn.resourcemanager.recovery.enabled</name>
    <value>true</value>
</property>
<property>
    <name>yarn.resourcemanager.store.class</name>
    <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
</property>
```

---

## 第5章 Hadoop 3.x HA 新特性 📝

### 5.1 支持多个 NameNode（>2）

Hadoop 3.x 支持配置 **3 个以上** NameNode，进一步提高可用性：

```xml
<property>
    <name>dfs.ha.namenodes.mycluster</name>
    <value>nn1,nn2,nn3</value>  <!-- 3个 NN -->
</property>
```

> 1 个 Active + 多个 Standby，任何一个 Standby 都可以快速接管。

### 5.2 Observer NameNode

Hadoop 3.x 新增 **Observer NameNode**：
- 处理**读请求**，分担 Active NN 压力
- 不能处理写请求
- 适合**读多写少**场景

---

## 🔥 面试高频题

### Q1：HDFS HA 的工作原理？
> 两个 NN，一个 Active 一个 Standby。通过 JournalNode 共享编辑日志实现元数据同步。ZKFC 监控 NN 健康状态，通过 ZooKeeper 选举实现自动故障转移。DN 同时向两个 NN 汇报 Block。

### Q2：如何防止脑裂？
> 通过 Fencing 机制：当一个 NN 变为 Active 前，先确保旧的 Active 被隔离（sshfence 杀进程或自定义 shell 脚本），防止两个 NN 同时 Active 导致数据不一致。

### Q3：JournalNode 的作用？需要几台？
> JournalNode 是 HA 架构中的共享存储系统，用于存放 Edits 编辑日志。Active NN 写入，Standby NN 读取。至少需要 3 台（2N+1），基于 Paxos 多数派写入保证一致性。

### Q4：HDFS HA 和 YARN HA 的区别？
> HDFS HA 需要 JournalNode 共享编辑日志 + ZKFC 故障转移。YARN HA 直接将 RM 状态存储到 ZK（ZKRMStateStore），由 ZK 选举 Active RM，不需要 JournalNode。
