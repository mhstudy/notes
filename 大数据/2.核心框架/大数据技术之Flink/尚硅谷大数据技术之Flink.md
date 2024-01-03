# 第1章 Flink概述

> Flink 官网：https://flink.apache.org/
>
> Flink 中文官网：<https://flink.apache.org/zh/>

![flink logo](https://flink.apache.org/img/logo/png/500/flink_squirrel_500.png)

## 1.1 Flink是什么

Flink核心目标，是“<font color = "red">数据流上的有状态计算</font>”（Stateful Computations over Data Streams）

具体说明：Apache Flink 是一个<font color = "red">框架</font>和<font color = "red">分布式处理引擎</font>
，用于在<font color = "red">无界</font>和<font color = "red">有界</font>数据流上进行<font color = "red">
有状态的计算</font>。

![流处理](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/flink-application-sources-sinks.png)

有界流和无界流

1) **无界数据流**：

* 有定义流的开始，但是没有定义流的结束：
* 它们会无休止的产生数据：
* 无界流的数据必须持续处理，即数据被摄取后需要立即处理。我们不能等到所有数据都到达再处理，因为输入时无限的。

2) **有界数据流**

* 有定义流的开始，也有定义流的结束：
* 有界流可以在摄取所有数据后再进行计算：
* 有界流所有数据可以被排序，所以并不需要有序摄取：
* 有界流处理通常被称为批处理。

有状态的流处理：

把流处理需要的<font color = "red">额外数据保存成一个“状态”</font>，然后针对这条数据进行处理，并且<font color = "red">
更新状态</font>。这就是所谓的“<font color = "red">有状态的流处理</font>”。

![有状态的流处理]()

* 状态在内存中：优点，速度快；缺点，可靠性差。
* 状态在分布式系统中：优点，可靠性高；缺点，速度慢

Flink的发展史：

## 1.2 Flink特点

我们处理数据的目标是：<font color = "red">低延迟、高吞吐、结果的准确性和良好的容错性</font>。

Flink主要特点如下：

* <font color = "red">高吞吐和低延迟</font>。每秒处理百万个事件，毫秒级延迟。
* <font color = "red">结果的准确性</font>。Flink提供了<font color = "red">事件时间</font>
  （event-time）和<font color = "red">处理时间</font>（processing-time）语义。对于乱序事件流，事件时间语义仍然能提供一致且准确的结果。
* <font color = "red">精确一次</font>（exactly-once）的状态一致性保证。
* <font color = "red">可连接到最常用的外部系统</font>，如Kafka、Hive、JDBC、HDFS、Redis等。
* <font color = "red">高可用</font>。本身高可用的设置，加上与K8S、YARN和Mesos的紧密集成，再加上从故障种快速恢复和动态扩展任务的能力，Flink能做到以极少的停机时间7×24全天侯运行。

## 1.3 Flink vs SparkStreaming

<font color = "red">Spark以批处理为根本</font>

* Spark数据模型：Spark采用RDD模型，Spark Streaming 的 DStream 实际上也就是一组组<font color = "red">
  小批数据RDD的集合</font>
* Spark运行时的架构：<font color = "red">Spark是批计算</font>，将DAG划分为不同的stage，<font color = "red">
  一个完成后才可以计算下一个</font>

<font color = "red">Flink以流处理为根本</font>

* Flink的数据模型：Flink 基本数据模型是<font color = "red">数据流</font>，以及事件（Event）序列
* Flink运行时的架构：Flink是标准的流执行模式，<font color = "red">
  一个事件在一个节点处理完后可以直接发往下一个节点进行处理</font>

![有界流和无界流](https://nightlies.apache.org/flink/flink-docs-master/fig/bounded-unbounded.png)

Flink和Spark Streaming 对比

|           | **Flink** | **Streaming**      |
|:---------:|-----------|--------------------|
| **计算模型**  | 流计算       | 微批处理               |
| **时间语义**  | 事件时间、处理时间 | 处理时间               |
|  **窗口**   | 多、灵活      | 少、不灵活（窗口必须是批次的整数倍） |
|  **状态**   | 有         | 没有                 |
| **流式SQL** | 有         | 没有                 |

## 1.4 Flink的应用场景

Flink在国内各个企业中大量使用。一些行业中的典型应用有：

1) 电商和市场营销。

举例：<font color = "red">实时</font>数据<font color = "red">报表</font>、<font color = "red">广告</font>
投放、<font color = "red">实时推荐</font>

2) 物联网（IOT）

举例：传感器实时数据采集和显示、<font color = "red">实时报警</font>，交通运输业

3) 物流配送和服务业

举例：<font color = "red">订单实时状态更新</font>、通知信息推送

4) 银行和金融业

举例：实时结算和通知推送，<font color = "red">实时检测异常行为</font>

## 1.5 Flink分层API

* 越顶层越抽象，表达含义越简明，使用越方便
* 越底层越具体，表达能力越丰富，使用越灵活

![Flink 分层API](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/levels_of_abstraction.svg)

有状态流处理：通过底层API（处理函数），对最原始数据加工处理。底层API与DataStream API相集成，可以处理复杂的计算。

<font color = "red">DataStream API（流处理）和DataSet API（批处理）</font>
封装了底层处理函数，提供了通用的模块，比如转换（transformations，包括map、flagmap等），连接（joins），聚合（aggregations），窗口（windows）操作等。注意：Flink1.12以后，<font color = "red">
DataStream API已经实现真正的流批一体，所以DataSetAPI已经过时。</font>

<font color = "red">Table API 是以表为中心的声明式编程</font>，其中表可能会动态变化。Table
API遵循关系模型：表有二维数据结构，类似于关系数据库中的表；同时API提供可比较的操作，例如<font color = "red">
select、project、join、group-by、aggregate</font>等。我们可以在表与 DataStream/DataSet 之间无缝切换，以允许程序将 Table API 与
DataStream 以及 DataSet 混合使用。

<font color = "red">SQL</font>这一层在语法与表达能力上与 Table API 类似，但是是<font color = "red">
以SQL查询表达式的形式表现程序</font>。SQL抽象与 Table API 交互密切，同时SQL查询可以直接在Table API定义的表上执行。

# 第2章 Flink快速上手

## 2.1 创建项目

在准备好所有的开发环境之后，我们就可以开始开发自己的第一个Flink程序了。首先我们要做的，就是在IDEA中搭建一个Flink项目的骨架。我们会使用Java项目中常见的Maven来进行依赖管理。

**1) 创建工程**

(1) 打开IntelliJ IDEA，创建一个Maven工程。

(2) 将这个Maven工程命名为 FlinkTutorial。

(3) 选定这个Maven工程所在存储路径，并点击Finish，Maven工程即创建成功。

**2) 添加项目依赖**

在项目的pom文件中，添加Flink的依赖，包括flink-java、flink-streaming-java，以及flink-clients（客户端，也可以省略）。

```xml
<properties>
    <flink.version>1.17.0</flink.version>
</properties>


  <dependencies>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-streaming-java</artifactId>
        <version>${flink.version}</version>
    </dependency>
    
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-clients</artifactId>
        <version>${flink.version}</version>
    </dependency>
  </dependencies>
```

## 2.2 WordCount代码编写

### 2.2.1 批处理

### 2.2.2 流处理

# 第3章 Flink部署

## 3.1 集群角色

![Flink 集群剖析](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/processes.svg)

## 3.2 Flink集群搭建

### 3.2.1 集群启动

### 3.2.2 向集群提交作业

## 3.3 部署模式

### 3.3.1 会话模式（Session Mode）

### 3.3.2 单作业模式（Per-Job Mode）

### 3.3.3 应用模式（Application Mode）

## 3.4 Standalone运行模式（了解）

### 3.4.1 会话模式部署

### 3.4.2 单作业模式部署

### 3.4.3 应用模式部署

## 3.5 YARN运行模式（重点）

### 3.5.1 相关准备和配置

### 3.5.2 会话模式部署

### 3.5.3 单作业模式部署

### 3.5.4 应用模式部署

## 3.6 K8S 运行模式（了解）

## 3.7 历史服务器

# 第4章 Flink运行时架构

## 4.1 系统架构

## 4.2 核心概念

### 4.2.1 并行度（Parallelism）

![并行度](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/parallel_dataflow.svg)

### 4.2.2 算子链（Operator Chain）

![合并算子链](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/tasks_chains.svg)

### 4.2.3 任务槽（Task Slots）

### 4.2.4 任务槽和并行度的关系

## 4.3 作业提交流程

### 4.3.1 Standalone会话模式作业提交流程

### 4.3.2 逻辑流图/作业图/执行图/物理流图

### 4.3.3 Yarn应用模式作业提交流程

# 第5章 DataStream API

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

## 5.3 转换算子（Transformation）

### 5.3.1 基本转换算子（map/ filter/ flatMap）

#### 5.3.1.1 映射（map）

#### 5.3.1.2 过滤（filter）

#### 5.3.1.3 扁平映射（flatMap）

### 5.3.2 聚合算子（Aggregation）

#### 5.3.2.1 按键分区（keyBy）

![keyBy](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/keyBy.png)

#### 5.3.2.2 简单聚合（sum/min/max/minBy/maxBy）

#### 5.3.2.3 归约聚合（reduce）

### 5.3.3 用户自定义函数（UDF）

#### 5.3.3.1 函数类（Function Classes）

#### 5.3.3.2 富函数类（Rich Function Classes）

### 5.3.4 物理分区算子（Physical Partitioning）

#### 5.3.4.1 随机分区（shuffle）

#### 5.3.4.2 轮询分区（Round-Robin）

#### 5.3.4.3 重缩放分区（rescale）

#### 5.3.4.4 广播（broadcast）

#### 5.3.4.5 全局分区（global）

#### 5.3.4.6 自定义分区（Custom）

### 5.3.5 分流

#### 5.3.5.1 简单实现

#### 5.3.5.2 使用侧输出流

### 5.3.6 基本合流操作

#### 5.3.6.1 联合（Union）

#### 5.3.6.2 连接（Connect）

![connect](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/connected-streams.svg)

## 5.4 输出算子（Sink）

### 5.4.1 连接到外部系统

### 5.4.2 输出到文件

### 5.4.3 输出到Kafka

### 5.4.4 输出到MySQL（JDBC）

### 5.4.5 自定义Sink输出

# 第6章 Flink中的时间和窗口

## 6.1 窗口（Window）

### 6.1.1 窗口的概念

### 6.1.2 窗口的分类

![windows](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/windows.svg)

![window-assigners](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/window-assigners.svg)

![tumbling-windows](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/tumbling-windows.svg)

![sliding-windows](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/sliding-windows.svg)

![session-windows](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/session-windows.svg)

![non-windowed](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/non-windowed.svg)

### 6.1.3 窗口API概览

### 6.1.4 窗口分配器

#### 6.1.4.1 时间窗口

#### 6.1.4.2 计数窗口

### 6.1.5 窗口函数

#### 6.1.5.1 增量聚合函数（ReduceFunction / AggregateFunction）

#### 6.1.5.2 全窗口函数（full window functions）

#### 6.1.5.3 增量聚合和全窗口函数的结合使用

### 6.1.6 其他API

#### 6.1.6.1 触发器（Trigger）

#### 6.1.6.2 移除器（Evictor）

## 6.2 时间语义

### 6.2.1 Flink中的时间语义

![Flink中的时间语义](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/event_processing_time.svg)

### 6.2.2 哪种时间语义更重要

## 6.3 水位线（Watermark）

### 6.3.1 事件时间和窗口

### 6.3.2 什么是水位线

### 6.3.3 水位线和窗口的工作原理

![stream_watermark_in_order](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/stream_watermark_in_order.svg)

![stream_watermark_out_of_order](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/stream_watermark_out_of_order.svg)

### 6.3.4 生成水位线

#### 6.3.4.1 生成水位线的总体原则

#### 6.3.4.2 水位线生成策略

#### 6.3.4.3 Flink内置水位线

#### 6.3.4.4 自定义水位线生成器

### 6.3.5 水位线的传递

### 6.3.6 迟到数据的处理

#### 6.3.6.1 推迟水印推进

#### 6.3.6.2 设置窗口延迟关闭

#### 6.3.6.3 使用侧流接收迟到的数据

## 6.4 基于时间的合流——双流联结（Join）

### 6.4.1 窗口联结（Window Join）

### 6.4.2 间隔联结（Interval Join）

# 第7章 处理函数

## 7.1 基本处理函数（ProcessFunction）

### 7.1.1 处理函数的功能和使用

### 7.1.2 ProcessFunction解析

### 7.1.3 处理函数的分类

## 7.2 按键分区处理函数（KeyedProcessFunction）

### 7.2.1 定时器（Timer）和定时服务（TimerService）

### 7.2.2 KeyedProcessFunction案例

## 7.3 窗口处理函数

### 7.3.1 窗口处理函数的使用

### 7.3.2 ProcessWindowFunction解析

## 7.4 应用案例——Top N

### 7.4.1 使用ProcessAllWindowFunction

### 7.4.2 使用KeyedProcessFunction

## 7.5 侧输出流（Side Output）

# 第8章 状态管理

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

# 第9章 容错机制

## 9.1 检查点（Checkpoint）

### 9.1.1 检查点的保存

### 9.1.2 从检查点恢复状态

### 9.1.3 检查点算法

#### 9.1.3.1 检查点分界线（Barrier）

#### 9.1.3.2 分布式快照算法（Barrier对齐的精准一次）

#### 9.1.3.3 分布式快照算法（Barrier对齐的至少一次）

#### 9.1.3.4 分布式快照算法（非Barrier对齐的精准一次）

### 9.1.4 检查点配置

#### 9.1.4.1 启用检查点

#### 9.1.4.2 检查点存储

#### 9.1.4.3 其它高级配置

#### 9.1.4.4 通用增量 checkpoint (changelog)

#### 9.1.4.5 最终检查点

### 9.1.5 保存点（Savepoint）

#### 9.1.5.1 保存点的用途

#### 9.1.5.2 使用保存点

#### 9.1.5.3 使用保存点切换状态后端

## 9.2 状态一致性

### 9.2.1 一致性的概念和级别

### 9.2.2 端到端的状态一致性

## 9.3 端到端精确一次（End-To-End Exactly-Once）

### 9.3.1 输入端保证

### 9.3.2 输出端保证

### 9.3.3 Flink和Kafka连接时的精确一次保证

# 第10章 Flink SQL

## 10.1 sql-client准备

### 10.1.1 基于yarn-session模式

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

### 10.5.1 With子句

### 10.5.2 SELECT & WHERE 子句

### 10.5.3 SELECT DISTINCT 子句

### 10.5.4 分组聚合

### 10.5.5 分组窗口聚合

### 10.5.6 窗口表值函数（TVF）聚合

### 10.5.7 Over 聚合

### 10.5.8 特殊语法 —— TOP-N

### 10.5.9 特殊语法 —— Deduplication去重

### 10.5.10 联结（Join）查询

### 10.5.11 Order by 和 limit

### 10.5.12 SQL Hints

### 10.5.13 集合操作

### 10.5.14 系统函数

### 10.5.15 Module操作

## 10.6 常用 Connector 读写

### 10.6.1 Kafka

### 10.6.2 File

### 10.6.3 JDBC（MySQL）

## 10.7 sql-client 中使用 savepoint

## 10.8 Catalog

### 10.8.1 Catalog类型

### 10.8.2 JdbcCatalog（MySQL）

### 10.8.3 HiveCatalog

## 10.9 代码中使用FlinkSQL

### 10.9.1 需要引入的依赖

### 10.9.2 创建表环境

### 10.9.3 创建表

### 10.9.4 表的查询

### 10.9.5 输出表

### 10.9.6 表和流的转换

### 10.9.7 自定义函数（UDF）


