# 第一章 Flink概述

## 1.1 Flink是什么

> Flink 官网：https://flink.apache.org/
>
> Flink 中文官网：<https://flink.apache.org/zh/>

![flink logo](https://flink.apache.org/img/logo/png/500/flink_squirrel_500.png)

## 1.2 Flink特点

## 1.3 Flink vs SparkStreaming

![有界流和无界流](https://nightlies.apache.org/flink/flink-docs-master/fig/bounded-unbounded.png)

|           | **Flink** | **Streaming**      |
|:---------:|-----------|--------------------|
| **计算模型**  | 流计算       | 微批处理               |
| **时间语义**  | 事件时间、处理时间 | 处理时间               |
|  **窗口**   | 多、灵活      | 少、不灵活（窗口必须是批次的整数倍） |
|  **状态**   | 有         | 没有                 |
| **流式SQL** | 有         | 没有                 |

## 1.4 Flink 应用场景

## 1.5 Flink 分层API

![Flink 分层API](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/levels_of_abstraction.svg)

# 第二章 Flink 快速上手

## 2.1 创建项目

## 2.2 WordCount代码编写

### 2.2.1 批处理

### 2.2.2 流处理

# 第三章 Flink 部署

## 3.1 集群角色

![Flink 集群剖析](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/processes.svg)

## 3.2  Flink 集群搭建

### 3.2.1 集群启动

### 3.2.2 向集群提交作业

## 3.3 部署模式

### 3.3.1 会话模式（Session Mode）

### 3.3.2 单作业模式（Per-Job Mode）

### 3.3.3 应用模式 （Application Mode）

## 3.4 Standalone 运行模式（了解）

### 3.4.1 会话模式部署

### 3.4.2 单作业模式部署

### 3.4.3 应用模式部署

## 3.5 YARN 运行模式（重点）

### 3.5.1 相关准备和配置

### 3.5.2 会话模式部署

### 3.5.3 单作业模式部署

### 3.5.4 应用模式部署

## 3.6 K8S 运行模式（了解）

## 3.7 历史服务器

# 第四章 Flink 运行时架构

## 4.1 系统架构

## 4.2 核心概念

### 4.2.1 并行度（Parallelism）

![并行度](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/parallel_dataflow.svg)

### 4.2.2 算子链（Operator Chain）

![合并算子链](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/tasks_chains.svg)

### 4.2.3 任务槽（Task Slots）

### 4.2.4 任务槽和并行度的关系

## 4.3 作业提交流程

### 4.3.1 Standalone 会话模式作业提交流程

### 4.3.2 逻辑流图/作业图/执行图/物理流图

### 4.3.3 Yarn 应用模式作业提交流程

# 第五章 DataStream API

## 5.1 执行环境（Execution Environment）

### 5.1.1 创建执行环境

### 5.1.2 执行模式（Execution Mode）

### 5.1.3 触发程序执行

## 5.2 源算子（Source）

### 5.2.1 准备工作

### 5.2.2 从集合中读取数据

### 5.2.3 从文件读取数据

### 5.2.4 从Socket读取数据

### 5.2.5 从Kafka读取数据

### 5.2.6 从数据生成器读取数据

### 5.2.7 Flink支持的数据类型

## 5.3 转换算子

### 5.3.1 基本转换算子（map/filter/flatmap）

### 5.3.2 聚合算子（Aggregation）

### 5.3.3 用户自定义函数（UDF）

### 5.3.4 物理分区算子（Physical Partitioning）

### 5.3.5 分流

### 5.3.6 基本合流操作

## 5.4 输出算子（Sink）

### 5.4.1 连接到外部系统

### 5.4.2 输出到文件

### 5.4.3 输出到Kafka

### 5.4.4 输出到MySQL（JDBC）

### 5.4.5 自定义Sink输出

# 第六章 Flink 中的时间和窗口

## 6.1 窗口（Window）

### 6.1.1 窗口的概念

### 6.1.2 窗口的分类

### 6.1.3 窗口API概览

### 6.1.4 窗口分配器

### 6.1.5 窗口函数

### 6.1.6 其他API

## 6.2 时间语义

### 6.2.1 Flink中的时间语义

### 6.2.2 哪种时间语义更重要

## 6.3 水位线（Watermark）

### 6.3.1 事件时间和窗口

### 6.3.2 什么是水位线

### 6.3.3 水位线和窗口的工作原理

### 6.3.4 生成水位线

### 6.3.5 水位线的传递

### 6.3.6 迟到数据的处理

## 6.4 基于时间的合流——双流联结（Join）

### 6.4.1 窗口联结（Window Join）

### 6.4.2 间隔联结（Interval Join）

# 第七章 处理函数

## 7.1 基本处理函数（ProcessFunction）

### 7.1.1 处理函数的功能和使用

### 7.1.2 ProcessFuntion解析

### 7.1.3 处理函数的分类

## 7.2 按键分区处理函数（KeyedProcessFuntion）

### 7.2.1 定时器（Timer）和定时服务（TimerServeice）

### 7.2.2 KeyedProcessFuntion案例

## 7.3 窗口处理函数

### 7.3.1 窗口处理函数的使用

### 7.3.2 ProcessWindowFunction解析

## 7.4 应用案例——TopN

### 7.4.1 使用ProcessAllWindowFunction

### 7.4.2 使用KededProcessFunction

## 7.5 侧输出流（Side Output）

# 第八章 状态管理

## 8.1 Flink中的状态

### 8.1.1 概述

### 8.1.2 状态的分类

## 8.2 按键分区状态（Keyed State）

### 8.2.1 值状态（ValueState）

### 8.2.2 列表状态（ListState）

### 8.2.3 Map状态（MapState）

### 8.2.4 归约状态（ReducingState）

### 8.2.5 聚合状态（AggregatingState）

### 8.2.6 状态生存时间（TTL）

## 8.3 算子状态（Operator State）

### 8.3.1 列表状态（ListState）

### 8.3.2 联合列表状态（UnionListState）

### 8.3.3 广播状态（BroadcastState）

## 8.4 状态后端（State Backends）

### 8.4.1 状态后端的分类（HashMapStateBackend/RocksDB）

### 8.4.2 如何选择正确的状态后端

### 8.4.3 状态后端的配置

# 第九章 容错机制

## 9.1 检查点（Checkpoint）

### 9.1.1 检查点的保存

### 9.1.2 从检查点恢复状态

### 9.1.3 检查点算法

### 9.1.4 检查点配置

### 9.1.5 保存点（Savepoint）

## 9.2 状态一致性

### 9.2.1 一致性的概念和级别

### 9.2.2 端到端的状态一致性

## 9.3 端到端精确一次（End-to-End Exactly-Once）

### 9.3.1 输入端的保证

### 9.3.2 输出端的保证

### 9.3.3 Flink和Kafka连接时的精确一次保证

# 第十章 Flink SQL

## 10.1 sql-client准备

### 10.1.1 基于yarn-session 模式

### 10.1.2 常用配置

## 10.2 流处理中的表

### 10.2.1 动态表和持续查询

### 10.2.2 将流转换成动态表

### 10.2.3 用SQL持续查询

### 10.2.4 将动态表转换为流

## 10.3 时间属性

### 10.3.1 事件时间

### 10.3.2 处理时间

## 10.4 DDL（Data Definition Language）数据定义

### 10.4.1 数据库

### 10.4.2 表

## 10.5 查询

### 10.5.0 DataGen & Print

### 10.5.1 With 子句

### 10.5.2 SELECT & WHERE 子句

### 10.5.3 SELECT & DISTINCT 子句

### 10.5.4 分组聚合

### 10.5.5 分组窗口聚合

### 10.5.6 窗口表值函数（TVF）聚合

### 10.5.7 Over 聚合

### 10.5.8 特殊语法——TOPN

### 10.5.9 特殊语法——Deduplication去重

### 10.5.10 联结（Join）查询

### 10.5.11 Order by 和 limit

### 10.5.12 SQL Hints

### 10.5.13 集合操作

### 10.5.14 系统函数

### 10.5.15 Module 操作

## 10.6 常用 Connector 读写

### 10.6.1 Kafka

### 10.6.2 File

### 10.6.3 JDBC（MySQL）

## 10.7 sql-client 中使用 save-point

## 10.8 Catalog

### 10.8.1 Catalog类型

### 10.8.2 JdbcCatalog（MySQL）

### 10.8.3 HiveCatalog

## 10.9 代码中使用FlinkSQL

### 10.9.1 需要引入的依赖

### 10.9.2 创建表环境
