#  SparkStreaming概述
##  Spark Streaming是什么
##  Spark Streaming的特点
##  Spark Streaming架构
###  架构图
### 1.3.2 背压机制
# 第2章 Dstream入门
## 2.1 WordCount案例实操
## 2.2 WordCount解析
# 第3章 DStream创建
## 3.1 RDD队列
### 3.1.1 用法及说明
### 3.1.2 案例实操
## 3.2 自定义数据源
### 3.2.1 用法及说明
### 3.2.2 案例实操
## 3.3 Kafka数据源（面试、开发重点）
### 3.3.1 版本选型
### 3.3.2 Kafka 0-8 Receiver模式（当前版本不适用）
### 3.3.3 Kafka 0-8 Direct模式（当前版本不适用）
### 3.3.4 Kafka 0-10 Direct模式
### 查看Kafka消费进度
# 第4章 DStream转换
## 4.1 无状态转化操作
### 4.1.1 Transform
### 4.1.2 join
## 4.2 有状态转化操作
### 4.2.1 UpdateStateByKey
### 4.2.2 WindowOperations
# 第5章 DStream输出
# 第6章 优雅关闭
# 第7章 SparkStreaming 案例实操
## 7.1 环境准备
### 7.1.1 pom文件
### 7.1.2 工具类
## 7.2 实时数据生成模块
## 7.3 需求一：广告黑名单
### 7.3.1 思路分析
### 7.3.2 MySQL建表
### 7.3.3 环境准备
### 7.3.4 代码实现
## 7.4 需求二：广告点击量实时统计
### 7.4.1 思路分析
### 7.4.2 MySQL建表
### 7.4.3 代码实现
## 7.5 需求三：最近一小时某个广告点击量趋势统计
### 7.5.1 思路分析
### 7.5.2 代码实现
