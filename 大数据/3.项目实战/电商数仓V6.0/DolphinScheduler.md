# DolphinScheduler

> Apache DolphinScheduler 官网：https://dolphinscheduler.apache.org/
> 官方文档：https://dolphinscheduler.apache.org/zh-cn/docs/
>
> 本文基于 DolphinScheduler 2.0.5 版本

---

## 第1章 DolphinScheduler 简介

### 1.1 🔥 概述

Apache DolphinScheduler 是一个**分布式、易扩展的可视化 DAG 工作流任务调度平台**。致力于解决数据处理流程中错综复杂的依赖关系，使调度系统在数据处理流程中**开箱即用**。

**核心特性：**
- 🔥 **DAG 可视化**：通过拖拽构建工作流，实时监控运行状态
- 🔥 **高可靠**：去中心化的多 Master 多 Worker 架构，支持 HA
- **丰富的任务类型**：Shell、SQL、Spark、Flink、MR、Python、HTTP 等
- **告警机制**：支持邮件、钉钉、微信等多种告警方式
- **资源管理**：支持 HDFS 资源中心
- **全局参数**：支持全局参数和本地参数传递

### 1.2 🔥 核心架构

| 组件 | 说明 |
|------|------|
| **MasterServer** | 🔥 负责 DAG 任务切分、任务提交监控、监听其他 Master 和 Worker 健康状态 |
| **WorkerServer** | 🔥 负责任务的执行和提供日志服务 |
| **ZooKeeper** | 注册中心，Master 和 Worker 通过 ZK 进行集群管理和容错 |
| **Alert** | 告警服务，提供告警相关接口 |
| **API** | Web 应用的 API 入口，提供 REST API |
| **UI** | 前端页面，提供可视化操作界面 |

**核心流程：**
```
用户 → API Server → MasterServer (DAG解析/任务分发)
                          ↓
                    ZooKeeper (注册/监听)
                          ↓
                    WorkerServer (任务执行)
                          ↓
                    Alert Server (告警通知)
```

### 1.3 ⭐ 调度器对比

| 对比项 | **DolphinScheduler** | **Azkaban** | **Airflow** | **Oozie** |
|--------|---------------------|-------------|-------------|-----------|
| 架构 | 🔥 去中心化 | 中心化 | 中心化 | 中心化 |
| 高可用 | ✅ 多 Master+Worker | ❌ 单点 | ❌ 单点 | ✅ |
| 任务定义 | 🔥 可视化拖拽 | JSON 配置 | Python 代码 | XML 配置 |
| 学习成本 | 低 | 低 | 高 | 高 |
| 社区 | Apache 顶级项目 | LinkedIn | 活跃 | 较弱 |

---

## 第2章 DolphinScheduler 部署

### 2.1 📝 环境要求

| 项目 | 要求 |
|------|------|
| **操作系统** | CentOS 7+、Ubuntu 18+ |
| **JDK** | JDK 1.8+ |
| **数据库** | MySQL 5.7+ / PostgreSQL 8.2+ |
| **ZooKeeper** | 3.4.6+ |
| **内存** | Master ≥ 4GB，Worker ≥ 4GB |

### 2.2 ⭐ 部署模式

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| **单机模式** | 所有服务部署在一台机器 | 快速体验 |
| **伪集群模式** | 所有服务部署在一台机器，但模拟集群 | 开发测试 |
| **🔥 集群模式** | 各服务分布式部署 | **生产环境** |

### 2.3 🔥 集群模式部署

#### 集群规划

| 主机 | Master | Worker | Alert | API | ZK |
|------|--------|--------|-------|-----|-----|
| hadoop102 | ✅ | ✅ | ✅ | ✅ | ✅ |
| hadoop103 | ✅ | ✅ | | | ✅ |
| hadoop104 | | ✅ | | | ✅ |

#### 关键配置

**install_config.conf：**

```bash
# 安装路径
installPath="/opt/module/dolphinscheduler"

# 部署用户
deployUser="atguigu"

# Master 服务器列表
masters="hadoop102,hadoop103"

# Worker 服务器列表（格式：主机名:分组名:权重）
workers="hadoop102:default:100,hadoop103:default:100,hadoop104:default:100"

# Alert 服务器
alertServer="hadoop102"

# API 服务器
apiServers="hadoop102"

# 数据库配置
DATABASE_TYPE="mysql"
SPRING_DATASOURCE_URL="jdbc:mysql://hadoop102:3306/dolphinscheduler?useUnicode=true&characterEncoding=UTF-8"
SPRING_DATASOURCE_USERNAME="dolphinscheduler"
SPRING_DATASOURCE_PASSWORD="dolphinscheduler123"

# ZooKeeper 配置
registryPluginName="zookeeper"
registryServers="hadoop102:2181,hadoop103:2181,hadoop104:2181"
```

#### 启停命令

```bash
# 一键部署
bash install.sh

# 启停命令
bash bin/start-all.sh    # 启动所有服务
bash bin/stop-all.sh     # 停止所有服务

# 单独启停
bash bin/dolphinscheduler-daemon.sh start master-server
bash bin/dolphinscheduler-daemon.sh start worker-server
bash bin/dolphinscheduler-daemon.sh start alert-server
bash bin/dolphinscheduler-daemon.sh start api-server

# Web UI 地址
# http://hadoop102:12345/dolphinscheduler
# 默认账号：admin / dolphinscheduler123
```

---

## 第3章 DolphinScheduler 入门

### 3.1 ⭐ 安全中心配置

**配置顺序**：创建租户 → 创建用户 → 创建告警组 → 配置 Worker 分组

| 概念 | 说明 |
|------|------|
| **租户** | 对应 Linux 用户，任务执行时使用该用户身份 |
| **用户** | 登录 DS 系统的用户，关联租户 |
| **告警组** | 告警通知的接收组 |
| **Worker 分组** | 将 Worker 分组，任务可指定在某个组执行 |
| **环境管理** | 配置任务运行的环境变量（如 JAVA_HOME、HADOOP_HOME） |

### 3.2 🔥 工作流管理

#### 工作流定义

通过**拖拽**创建 DAG 工作流：

1. 创建项目 → 进入项目 → 工作流定义 → 创建工作流
2. 拖拽任务节点到画布
3. 连接节点（定义依赖关系）
4. 配置每个任务节点的参数
5. 保存工作流

#### 🔥 常用任务类型

| 任务类型 | 说明 | 典型使用 |
|----------|------|----------|
| **Shell** | 🔥 执行 Shell 脚本 | 数据采集脚本、Hadoop 命令 |
| **SQL** | 执行 SQL 语句 | Hive SQL、MySQL 查询 |
| **Spark** | 提交 Spark 任务 | Spark ETL |
| **Flink** | 提交 Flink 任务 | 实时计算 |
| **MapReduce** | 提交 MR 任务 | 离线计算 |
| **Python** | 执行 Python 脚本 | 数据处理 |
| **Dependent** | 🔥 依赖检查 | 跨工作流依赖 |
| **Sub Process** | 子工作流 | 工作流复用 |
| **HTTP** | HTTP 请求 | API 调用 |

#### 执行工作流

**运行策略**：
- **并行**：同一工作流可同时运行多个实例
- **串行等待**：同一工作流排队等待
- **串行抢占**：新实例抢占旧实例
- **串行丢弃**：有运行中实例时丢弃新实例

**定时调度**：使用 Cron 表达式
```
# 每天凌晨 1 点执行
0 0 1 * * ? *

# 每小时执行一次
0 0 * * * ? *

# 每周一凌晨 2 点执行
0 0 2 ? * MON *
```

---

## 第4章 DolphinScheduler 进阶

### 4.1 🔥 工作流传参

#### 全局参数与本地参数

```bash
# 全局参数（工作流级别）
# 在工作流定义中设置：
# dt = ${system.biz.date}    # 业务日期（T-1）

# 本地参数（任务级别）
# 在 Shell 任务中使用：
echo "当前业务日期: ${dt}"
```

#### 🔥 内置参数

| 参数 | 说明 | 示例 |
|------|------|------|
| `${system.biz.date}` | 🔥 业务日期（调度日期-1天） | 20240101 |
| `${system.biz.curdate}` | 调度日期 | 20240102 |
| `${system.datetime}` | 调度时间（精确到秒） | 20240102120000 |

**日期加减：**
```
$[yyyy-MM-dd-1]    # 调度日期 - 1天
$[yyyyMMdd+7]      # 调度日期 + 7天
$[yyyy-MM-01]      # 调度日期所在月的1号
```

#### 参数优先级

🔥 **本地参数 > 上游任务传递参数 > 全局参数 > 内置参数**

### 4.2 ⭐ 引用依赖资源

- 在**资源中心**上传脚本、配置文件等
- 任务节点中可引用资源中心的文件
- 支持 HDFS / S3 / 本地文件系统

### 4.3 ⭐ 告警通知

- 成功时发送 / 失败时发送 / 成功和失败都发送
- 支持邮件、钉钉、微信企业号、HTTP 等
- 在告警实例管理中配置告警插件

### 4.4 📝 工作流失败重跑

- **从失败节点开始重跑**：跳过已成功的节点
- **从当前节点开始重跑**：手动指定从哪个节点开始
- **恢复暂停**：继续执行暂停的工作流

---

## 🔥 DolphinScheduler 面试高频问题

### Q1：DolphinScheduler 的架构和核心组件？

- **去中心化架构**：多 Master + 多 Worker，通过 ZK 进行服务注册和发现
- Master：DAG 解析、任务分发、容错监控
- Worker：任务执行，向 Master 汇报状态
- 🔥 Master 和 Worker 都支持**横向扩展**

### Q2：每天集群运行多少指标？任务挂了怎么办？

- 每天运行指标数取决于业务需求，通常 **100~500 个任务**
- 任务失败处理：
  1. 查看任务日志定位问题
  2. 修复后从**失败节点重跑**
  3. 设置**失败重试次数**（通常 2~3 次）
  4. 配置**告警通知**，第一时间感知

### Q3：DS 挂了怎么办？

- 🔥 **Master HA**：多 Master 部署，一个挂了其他自动接管
- 🔥 **Worker HA**：多 Worker 部署，任务会重新分配到其他 Worker
- ZK 负责故障检测和服务发现
- Master 挂掉后，其负责的工作流实例会被其他 Master **容错接管**
