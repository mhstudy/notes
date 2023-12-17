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

# 第三章 Flink 部署

## 3.1 集群角色

![Flink 集群剖析](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/processes.svg)

# 第四章 Flink 运行时架构

## 4.1 系统架构

## 4.2 核心概念

### 4.2.1 并行度（Parallelism）

![并行度](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/parallel_dataflow.svg)

### 4.2.2 算子链（Operator Chain）

![合并算子链](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/tasks_chains.svg)

### 4.2.3 任务槽（Task Slots）

### 4.2.3 任务槽和并行度的关系

## 4.3 作业提交流程

# 第五章 DataStream API

# 第六章 Flink 中的时间和窗口

# 第七章 处理函数

# 第八章 状态管理

# 第九章 容错机制

# 第十章 Flink SQL