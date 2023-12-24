#  Spark概述
##  Spark是什么
##  Spark and Hadoop
##  Spark or Hadoop
##  Spark 核心模块
#  Spark快速上手
## 2.1  创建Maven项目
### 2.1.1 增加Scala插件
### 2.1.2 增加依赖关系
### 2.1.3 WordCount
### 2.1.4 异常处理
#  Spark运行环境
## 3.1  Local模式
### 3.1.1 解压缩文件
### 3.1.2 启动Local环境
### 3.1.3 命令行工具
### 3.1.4 退出本地模式
### 3.1.5 提交应用
## 3.2  Standalone模式
### 3.2.1 解压缩文件
### 3.2.2 修改配置文件
### 3.2.3 启动集群
### 3.2.4 提交应用
### 3.2.5 提交参数说明
### 3.2.6 配置历史服务
### 3.2.7 配置高可用（HA）
## 3.3  Yarn模式
### 3.3.1 解压缩文件
### 3.3.2 修改配置文件
### 3.3.3 启动HDFS以及YARN集群
### 3.3.4 提交应用
### 3.3.5 配置历史服务器
## 3.4  K8S & Mesos模式
## 3.5  Windows模式
### 3.5.1 解压缩文件
### 3.5.2 启动本地环境
### 3.5.3 命令行提交应用
## 3.6  部署模式对比
## 3.7  端口号
#  Spark运行架构
## 4.1 运行架构
## 4.2 核心组件
### 4.2.1 Driver
### 4.2.2 Executor
### 4.2.3 Master & Worker
### 4.2.4 ApplicationMaster
## 4.3 核心概念
### 4.3.1 Executor与Core（核）
### 4.3.2 并行度（Parallelism）
### 4.3.3 有向无环图（DAG）
## 4.4 提交流程
### 4.2.1 Yarn Client模式
### 4.2.2 Yarn Cluster模式
#  Spark核心编程
## 5.1 RDD
### 5.1.1 什么是RDD
### 5.1.2 核心属性
### 5.1.3 执行原理
### 5.1.4 基础编程
#### 5.1.4.1 RDD创建
#### 5.1.4.2 RDD并行度与分区
#### 5.1.4.3 RDD转换算子
##### map
##### mapPartitions
##### mapPartitionsWithIndex
##### flatMap
##### glom
##### groupBy
##### filter
##### sample
##### distinct
##### coalesce
##### repartition
##### sortBy
##### intersection
##### union
##### subtract
##### zip
##### partitionBy
##### reduceByKey
##### groupByKey
##### aggregateByKey
##### foldByKey
##### combineByKey
##### sortByKey
##### join
##### leftOuterJoin
##### cogroup
#### 5.1.4.4 案例实操
#### 5.1.4.5 RDD行动算子
##### reduce
##### collect
##### count
##### first
##### take
##### takeOrdered
##### aggregate
##### fold
##### countByKey
##### save相关算子
##### foreach
#### 5.1.4.6 RDD序列化
#### 5.1.4.7 RDD依赖关系
#### 5.1.4.8 RDD持久化
#### 5.1.4.9 RDD分区器
#### 5.1.4.10 RDD文件读取与保存
## 5.2 累加器
### 5.2.1 实现原理
### 5.2.2 基础编程
#### 5.2.2.1 系统累加器
#### 5.2.2.2 自定义累加器
## 5.3 广播变量
### 5.3.1 实现原理
### 5.3.2 基础编程
#  Spark案例实操
## 6.1 需求1：Top10热门品类
### 6.1.1 需求说明
### 6.1.2 实现方案一
#### 6.1.2.1 需求分析
#### 6.1.2.2 需求实现
### 6.1.3 实现方案二
#### 6.1.3.1 需求分析
#### 6.1.3.2 需求实现
### 6.1.4 实现方案三
#### 6.1.4.1 需求分析
#### 6.1.4.2 需求实现
## 6.2 需求2：Top10热门品类中每个品类的Top10活跃Session统计
#### 6.2.1 需求说明
### 6.2.2 需求分析
### 6.2.3 功能实现
## 6.3 需求3：页面单跳转换率统计
### 6.3.1 需求说明
### 6.3.2 需求分析
### 6.3.3 功能实现