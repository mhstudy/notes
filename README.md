# 📚 大数据全栈学习笔记

> 🔗 在线访问：https://zhouminghan.github.io/notes/

## 📖 笔记导读

本站是一套**系统化的大数据技术栈学习笔记**，涵盖从基础到实战的完整知识体系。知识点分为三个层级：

| 标记 | 含义 | 说明 |
|:---:|:---:|:---|
| 🔥 | **面试高频 / 生产常用** | 详细讲解 + 代码案例 + 面试问答 |
| ⭐ | **重要知识点** | 核心概念需理解，配简要代码 |
| 📝 | **了解即可** | 简要说明，知道即可 |

---

## 🗺️ 大数据技术栈全景图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        📊 大数据技术栈全景                               │
├─────────────┬──────────────┬──────────────┬──────────────┬─────────────┤
│  数据采集    │   数据存储    │   数据计算    │   数据查询    │  任务调度    │
├─────────────┼──────────────┼──────────────┼──────────────┼─────────────┤
│ Flume       │ HDFS         │ MapReduce    │ Hive         │ DolphinSch. │
│ Maxwell     │ HBase        │ Spark        │ ClickHouse   │ Azkaban     │
│ DataX       │ Kafka        │ Flink        │ Doris        │ Oozie       │
│ FlinkCDC    │ Redis        │ Spark SQL    │ Iceberg      │             │
│ Canal       │ Iceberg      │ Spark Stream │ Presto       │             │
├─────────────┼──────────────┼──────────────┼──────────────┼─────────────┤
│  编程语言    │  资源调度     │   协调服务    │   数据治理    │  可视化      │
├─────────────┼──────────────┼──────────────┼──────────────┼─────────────┤
│ Java        │ YARN         │ Zookeeper    │ Atlas        │ Superset    │
│ Scala       │ Kubernetes   │              │ Griffin      │ Grafana     │
│ Python      │ Mesos        │              │ Ranger       │ DataV       │
│ SQL         │              │              │              │             │
├─────────────┴──────────────┴──────────────┴──────────────┴─────────────┤
│                     🛠️ 基础设施：Linux · Shell · Git · Maven            │
└─────────────────────────────────────────────────────────────────────────┘
```

> 📌 **参考资源**：
> - [Apache 软件基金会](https://www.apache.org/) — 大数据核心项目的开源社区
> - [DB-Engines 数据库排名](https://db-engines.com/en/ranking) — 数据库技术流行度追踪
> - [Hadoop 官方生态](https://hadoop.apache.org/) — 分布式计算基石

---

## 📊 大数据学习路线

### 1️⃣ 核心基础

| 技术 | 说明 | 重要度 |
|:---|:---|:---:|
| [Linux & Shell](大数据/1.核心基础/Linux/Linux.md) | 服务器管理 & Shell 脚本 | ⭐ |
| [Java SE](大数据/1.核心基础/Java核心基础/Java_SE基础篇.md) | Java 核心编程 | ⭐ |
| [MySQL](大数据/1.核心基础/MySQL/MySQL基础篇.md) | 关系型数据库基础 | ⭐ |
| [JDBC](大数据/1.核心基础/JDBC/JDBC.md) | 数据库连接技术 | ⭐ |
| [Maven](大数据/1.核心基础/Maven/Maven.md) | 项目构建管理 | ⭐ |
| [Git](大数据/1.核心基础/Git/Git.md) | 版本控制 | ⭐ |
| [Scala](大数据/1.核心基础/Scala/Scala.md) | Spark 开发语言 | ⭐ |
| [Python](大数据/1.核心基础/Python/Python.md) | 数据分析 & PySpark | 📝 |

### 2️⃣ 核心框架（重点！）

| 技术 | 说明 | 重要度 |
|:---|:---|:---:|
| [Hadoop](大数据/2.核心框架/Hadoop/01_大数据概论.md) | 分布式存储与计算基石 | 🔥 |
| [Zookeeper](大数据/2.核心框架/Zookeeper/Zookeeper.md) | 分布式协调服务 | 🔥 |
| [Hive](大数据/2.核心框架/Hive/Hive.md) | 数据仓库工具（SQL on Hadoop）| 🔥 |
| [HBase](大数据/2.核心框架/HBase/HBase.md) | 分布式列存数据库 | 🔥 |
| [Kafka](大数据/2.核心框架/Kafka/01_Kafka核心.md) | 分布式消息队列 | 🔥 |
| [Spark](大数据/2.核心框架/Spark/01_SparkCore.md) | 内存计算引擎 | 🔥 |
| [Flink](大数据/2.核心框架/Flink/Flink.md) | 流批一体计算引擎 | 🔥 |

### 3️⃣ 数据采集与同步

| 技术 | 说明 | 重要度 |
|:---|:---|:---:|
| [Flume](大数据/2.核心框架/Flume/Flume.md) | 日志采集系统 | 🔥 |
| [DataX](大数据/2.核心框架/DataX/DataX.md) | 离线数据同步 | ⭐ |
| [Maxwell](大数据/2.核心框架/Maxwell/Maxwell.md) | MySQL Binlog 增量同步 | ⭐ |
| [FlinkCDC](大数据/2.核心框架/FlinkCDC/FlinkCDC.md) | 实时变更数据捕获 | 🔥 |

### 4️⃣ OLAP 分析引擎

| 技术 | 说明 | 重要度 |
|:---|:---|:---:|
| [ClickHouse](大数据/2.核心框架/ClickHouse/ClickHouse.md) | 列式分析数据库 | 🔥 |
| [Doris](大数据/2.核心框架/Doris/Doris.md) | MPP 分析数据库 | ⭐ |
| [Iceberg](大数据/2.核心框架/Iceberg/Iceberg.md) | 数据湖表格式 | ⭐ |

### 5️⃣ 其他组件

| 技术 | 说明 | 重要度 |
|:---|:---|:---:|
| [Redis](大数据/2.核心框架/Redis/Redis.md) | 缓存数据库 | 🔥 |
| [DolphinScheduler](大数据/3.项目实战/电商数仓V6.0/DolphinScheduler.md) | 任务调度 | ⭐ |

### 6️⃣ 项目实战

- 📦 [电商数仓 V6.0 全流程](大数据/3.项目实战/电商数仓V6.0/电商数仓（4电商数据仓库系统）.md)

### 🔥 面试突击

- 📋 [大数据高频面试题](大数据/高频面试题.md)
- 📖 [技术总览](大数据/技术总览.md)

---

## 🛠️ 开发工具

| 工具 | 链接 |
|:---|:---|
| Jetbrains 快捷键 | [查看](开发工具/Jetbrains系列常用快捷键.md) |
| Sublime Text | [查看](开发工具/SublimeText使用.md) |
| Docsify 搭建 | [查看](开发工具/docsify.md) |

---

## 🎯 面试有招

| 主题 | 链接 |
|:---|:---|
| IT 行业求职指导 | [查看](面试有招/明哥聊求职/IT行业求职指导.md) |
| 求职指导测试题 | [查看](面试有招/明哥聊求职/尚硅谷求职指导课程测试题.md) |

---

> 💡 **Tips**：使用左上角搜索框可快速定位笔记内容，使用 `Ctrl + K` 快捷键打开搜索
