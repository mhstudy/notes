# Hadoop 入门

> 🔗 **官方网站**：https://hadoop.apache.org/
> 📖 **官方文档**：https://hadoop.apache.org/docs/stable/
> 📌 **学习版本**：Hadoop 3.3.x

---

---

## 第1章 Hadoop 概述 ⭐

### 1.1 Hadoop 是什么

Hadoop 是 Apache 基金会开发的**分布式系统基础架构**，主要解决海量数据的**存储**和**计算**问题。

> 广义上 Hadoop 指整个 Hadoop 生态圈（HBase、Hive、Spark 等）

### 1.2 Hadoop 发展历史 📝

- 2003-2004：Google 发表 GFS、MapReduce、BigTable 论文
- 2006：Doug Cutting 创建 Hadoop，以儿子的玩具象命名
- 2008：Hadoop 成为 Apache 顶级项目
- 2012：Hadoop 2.x（引入 YARN）
- 2017：Hadoop 3.x（纠删码、多 NN 支持等）

### 1.3 Hadoop 三大发行版本 📝

| 发行版 | 说明 | 适用场景 |
|:---|:---|:---|
| **Apache** | 原生开源，需自行配置 | 学习、定制化需求 |
| **CDH（Cloudera）** | 集成度高，有管理界面 | 企业生产（大公司） |
| **HDP（Hortonworks）** | 开源，已与 Cloudera 合并 | 企业（已被 CDP 替代） |

### 1.4 Hadoop 优势（4高）🔥

| 优势 | 说明 |
|:---|:---|
| **高可靠性** | 数据多副本存储（默认 3 副本），自动容错 |
| **高扩展性** | 节点动态增减，线性扩展 |
| **高效性** | MapReduce 并行计算，加快处理速度 |
| **高容错性** | 任务失败自动重新分配 |

### 1.5 Hadoop 组成（面试重点）🔥🔥

```
Hadoop 1.x                    Hadoop 2.x / 3.x
┌─────────────────┐           ┌─────────────────┐
│   MapReduce     │           │   MapReduce     │  ← 计算
│ (计算 + 资源调度) │           ├─────────────────┤
├─────────────────┤           │     YARN        │  ← 资源调度
│     HDFS        │           ├─────────────────┤
│   (数据存储)     │           │     HDFS        │  ← 存储
└─────────────────┘           └─────────────────┘
```

> 💡 **面试**：Hadoop 1.x vs 2.x 最大区别？  
> **答**：2.x 将资源调度从 MapReduce 剥离出来形成独立的 **YARN**，使 Spark/Flink 等计算框架也可运行在 Hadoop 上。

#### 1.5.1 HDFS 架构概述

| 组件 | 说明 |
|:---|:---|
| **NameNode** | 存储文件元数据（文件名、目录、属性、块列表、块所在 DN） |
| **DataNode** | 存储实际数据块（Block） |
| **Secondary NameNode** | 辅助 NN 进行 Fsimage 和 Edits 合并 |

#### 1.5.2 YARN 架构概述

| 组件 | 说明 |
|:---|:---|
| **ResourceManager** | 整个集群资源的管理者 |
| **NodeManager** | 单个节点资源的管理者 |
| **ApplicationMaster** | 单个任务的管理者 |
| **Container** | 资源抽象（CPU、内存） |

#### 1.5.3 MapReduce 架构概述

- **Map 阶段**：并行处理输入数据
- **Reduce 阶段**：对 Map 结果汇总

#### 1.5.4 HDFS、YARN、MapReduce 三者关系 🔥

```
Client 提交 Job
      │
      ▼
    YARN (RM)  ──→  分配 Container  ──→  NodeManager
      │                                      │
      ▼                                      ▼
  ApplicationMaster                    MapTask / ReduceTask
      │                                  ↕ (读写数据)
      └────────────────────────────→    HDFS
```

---

## 第2章 Hadoop 运行环境搭建 🔥

### 2.1 集群规划

| | hadoop102 | hadoop103 | hadoop104 |
|:---|:---|:---|:---|
| **HDFS** | NameNode、DataNode | DataNode | SecondaryNameNode、DataNode |
| **YARN** | NodeManager | ResourceManager、NodeManager | NodeManager |

### 2.2 核心配置文件 🔥

**4 个核心配置文件**（`$HADOOP_HOME/etc/hadoop/`）：

| 配置文件 | 说明 |
|:---|:---|
| `core-site.xml` | Hadoop 核心配置（NN 地址等） |
| `hdfs-site.xml` | HDFS 配置（副本数等） |
| `yarn-site.xml` | YARN 配置（RM 地址等） |
| `mapred-site.xml` | MapReduce 配置 |

```xml
<!-- core-site.xml -->
<configuration>
    <!-- NameNode 地址 -->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop102:8020</value>
    </property>
    <!-- Hadoop 数据存储目录 -->
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/opt/module/hadoop-3.3.4/data</value>
    </property>
</configuration>
```

```xml
<!-- hdfs-site.xml -->
<configuration>
    <!-- NN Web 端访问地址 -->
    <property>
        <name>dfs.namenode.http-address</name>
        <value>hadoop102:9870</value>
    </property>
    <!-- 2NN Web 端访问地址 -->
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>hadoop104:9868</value>
    </property>
</configuration>
```

### 2.3 常用端口号 🔥🔥

| 端口号 | 说明 | Hadoop 版本 |
|:---|:---|:---|
| **9870** | HDFS NameNode Web UI | 3.x |
| **8088** | YARN ResourceManager Web UI | 2.x/3.x |
| **8020** | NN 内部通信端口（RPC） | 3.x |
| **19888** | 历史服务器 Web UI | 2.x/3.x |

> ⚠️ Hadoop 2.x 中 NN Web 端口为 **50070**

### 2.4 集群启停命令 🔥

```bash
# 整体启停
start-dfs.sh / stop-dfs.sh      # 启停 HDFS
start-yarn.sh / stop-yarn.sh    # 启停 YARN

# 单个组件
hdfs --daemon start namenode     # 启动 NameNode
hdfs --daemon start datanode     # 启动 DataNode
yarn --daemon start resourcemanager
yarn --daemon start nodemanager

# 历史服务器
mapred --daemon start historyserver
```

### 2.5 集群常用脚本 ⭐

```bash
#!/bin/bash
# myhadoop.sh — 一键启停 Hadoop 集群
if [ $# -lt 1 ]; then
    echo "Usage: myhadoop.sh start|stop"
    exit
fi

case $1 in
"start")
    echo "========== 启动 Hadoop 集群 =========="
    echo "---------- 启动 HDFS ----------"
    ssh hadoop102 "/opt/module/hadoop-3.3.4/sbin/start-dfs.sh"
    echo "---------- 启动 YARN ----------"
    ssh hadoop103 "/opt/module/hadoop-3.3.4/sbin/start-yarn.sh"
    echo "---------- 启动 HistoryServer ----------"
    ssh hadoop102 "/opt/module/hadoop-3.3.4/bin/mapred --daemon start historyserver"
    ;;
"stop")
    echo "========== 关闭 Hadoop 集群 =========="
    echo "---------- 关闭 HistoryServer ----------"
    ssh hadoop102 "/opt/module/hadoop-3.3.4/bin/mapred --daemon stop historyserver"
    echo "---------- 关闭 YARN ----------"
    ssh hadoop103 "/opt/module/hadoop-3.3.4/sbin/stop-yarn.sh"
    echo "---------- 关闭 HDFS ----------"
    ssh hadoop102 "/opt/module/hadoop-3.3.4/sbin/stop-dfs.sh"
    ;;
*)
    echo "Usage: myhadoop.sh start|stop"
    ;;
esac
```

---

## 🔥 面试高频题

### Q1：Hadoop 的组成模块？
> HDFS（存储）、YARN（资源调度）、MapReduce（计算）。Hadoop 2.x 最大变化是将资源调度从 MR 剥离为 YARN。

### Q2：Hadoop 常用端口号？
> HDFS Web UI: 9870（3.x）/ 50070（2.x）；YARN Web UI: 8088；NN RPC: 8020。

### Q3：Hadoop 配置文件有哪些？
> 4个核心配置：`core-site.xml`、`hdfs-site.xml`、`yarn-site.xml`、`mapred-site.xml`。此外还有 `workers`（配置 DataNode 列表）和 `hadoop-env.sh`（环境变量）。
