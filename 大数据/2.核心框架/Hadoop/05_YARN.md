# YARN

> 🔗 **官方文档**：https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/YARN.html
> 📌 **学习版本**：Hadoop 3.3.x

---

## 第1章 YARN 概述 ⭐

### 1.1 什么是 YARN

YARN（Yet Another Resource Negotiator）是 Hadoop 的**资源管理和作业调度平台**。

### 1.2 YARN 架构 🔥🔥

![YARN架构](https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/yarn_architecture.gif ':size=600')

| 组件 | 说明 |
|:---|:---|
| **ResourceManager（RM）** | 集群资源管理者，负责调度和分配资源 |
| **NodeManager（NM）** | 每个节点的资源管理者，汇报资源使用 |
| **ApplicationMaster（AM）** | 每个应用的管理者，负责任务调度和监控 |
| **Container** | 资源的抽象封装（CPU + 内存） |

### 1.3 YARN 工作流程 🔥🔥

```
1. Client 提交作业到 RM
2. RM 分配一个 Container 启动 AM
3. AM 向 RM 申请资源（Container）
4. RM 将资源分配给 NM 上的 Container
5. AM 在 Container 中启动 Task（Map/Reduce）
6. Task 执行完毕，AM 汇报结果给 RM
7. RM 回收资源
```

---

## 第2章 调度器 🔥🔥

| 调度器 | 说明 | 适用场景 |
|:---|:---|:---|
| **FIFO** | 先进先出 | 测试（不推荐） |
| **Capacity（容量）** 🔥 | 多队列，每队列有容量保证 | **Apache 默认**，多租户 |
| **Fair（公平）** 🔥 | 多队列，资源公平共享 | **CDH 默认** |

### Capacity 调度器配置

```xml
<!-- capacity-scheduler.xml -->
<property>
    <name>yarn.scheduler.capacity.root.queues</name>
    <value>default,hive,spark</value>
</property>
<property>
    <name>yarn.scheduler.capacity.root.default.capacity</name>
    <value>40</value>
</property>
<property>
    <name>yarn.scheduler.capacity.root.hive.capacity</name>
    <value>40</value>
</property>
<property>
    <name>yarn.scheduler.capacity.root.spark.capacity</name>
    <value>20</value>
</property>
```

### Capacity vs Fair 详细对比 🔥

| 对比维度 | Capacity 调度器 | Fair 调度器 |
|:---|:---|:---|
| 资源保证 | 按队列预留最小资源 | 动态均分，空闲资源可借 |
| 资源抢占 | 不支持 | ✅ 支持资源抢占 |
| 调度策略 | FIFO（队列内） | 公平/FIFO/DRF 可选 |
| 默认使用 | Apache Hadoop | CDH |
| 多租户 | ✅ | ✅ |

---

## 第3章 核心参数 ⭐

### 3.1 ResourceManager 参数

| 参数 | 默认 | 说明 |
|:---|:---|:---|
| `yarn.resourcemanager.scheduler.class` | CapacityScheduler | 调度器类名 |
| `yarn.resourcemanager.max-completed-applications` | 10000 | 保存的已完成应用数量 |

### 3.2 NodeManager 参数

| 参数 | 默认 | 说明 |
|:---|:---|:---|
| `yarn.nodemanager.resource.memory-mb` | 8192MB | NM 可分配的总内存 |
| `yarn.nodemanager.resource.cpu-vcores` | 8 | NM 可分配的总 CPU 核数 |
| `yarn.nodemanager.vmem-pmem-ratio` | 2.1 | 虚拟内存与物理内存比 |

### 3.3 Container 参数

| 参数 | 默认 | 说明 |
|:---|:---|:---|
| `yarn.scheduler.minimum-allocation-mb` | 1024MB | Container 最小内存 |
| `yarn.scheduler.maximum-allocation-mb` | 8192MB | Container 最大内存 |
| `yarn.scheduler.minimum-allocation-vcores` | 1 | Container 最小核数 |
| `yarn.scheduler.maximum-allocation-vcores` | 4 | Container 最大核数 |

---

## 第4章 YARN 常用命令 ⭐

```bash
# 查看所有应用
yarn application -list

# 查看指定状态的应用
yarn application -list -appStates RUNNING

# 杀死应用
yarn application -kill <application_id>

# 查看应用日志
yarn logs -applicationId <application_id>

# 查看集群节点状态
yarn node -list

# 查看队列信息
yarn queue -status default

# 更新队列配置（刷新）
yarn rmadmin -refreshQueues
```

---

## 第5章 任务推测执行 ⭐

**推测执行（Speculative Execution）**：当发现某个 Task 比其他 Task 慢很多时，YARN 会在另一个节点上启动一个**备份 Task**，哪个先完成就用哪个的结果。

```xml
<!-- mapred-site.xml -->
<property>
    <name>mapreduce.map.speculative</name>
    <value>true</value>
</property>
<property>
    <name>mapreduce.reduce.speculative</name>
    <value>true</value>
</property>
```

> ⚠️ **注意**：在数据倾斜严重时，推测执行可能会**浪费资源**，应考虑关闭。

---

## 第6章 面试题 🔥🔥🔥

### Q1：YARN 的工作流程？
> Client → RM → AM → RM 申请资源 → NM 启动 Container → Task 执行 → 结果返回

### Q2：Capacity 和 Fair 调度器的区别？
> Capacity：按队列预留资源，保证最小容量。Fair：动态均分资源，空闲资源可被其他队列使用。

### Q3：如何指定提交队列？
> `set mapreduce.job.queuename=hive;` 或 Spark 中 `--queue spark`

### Q4：YARN 如何处理 Container 失败？
> AM 检测到 Container 失败后，向 RM 重新申请资源启动新的 Container。如果 AM 自身失败，RM 会重新启动 AM（默认最多重试 2 次）。

### Q5：Container 的内存和 CPU 如何配置？
> 根据集群物理资源配置：`yarn.nodemanager.resource.memory-mb` 设为物理内存的 80%~85%，`yarn.nodemanager.resource.cpu-vcores` 设为物理核数的 80%~85%。

### Q6：什么是 YARN 的推测执行？
> 当某个 Task 运行特别慢时，YARN 在另一个节点启动备份 Task，谁先完成用谁的结果。注意：数据倾斜场景下应关闭推测执行。
