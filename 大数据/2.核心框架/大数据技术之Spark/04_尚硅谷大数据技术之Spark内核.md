# 第1章Spark内核概述
## 1.1 Spark核心组件回顾
### 1.1.1 Driver
### 1.1.2 Executor
## 1.2 Spark通用运行流程概述
# 第2章 Spark部署模式
## 2.1 YARN模式运行机制
### 2.1.1 YARN Cluster模式
### 2.1.2 YARN Client模式
## 2.2 Standalone模式运行机制
### 2.2.1 Standalone Cluster模式
### 2.2.2 Standalone Client模式
# 第3章 Spark通讯架构
## 3.1 Spark通信架构概述
## 3.2 Spark通讯架构解析
# 第4章 Spark任务调度机制
## 4.1 Spark任务调度概述
## 4.2 Spark Stage级调度
## 4.3 Spark Task级调度
### 4.3.1 调度策略
### 4.3.2 本地化调度
### 4.3.3 失败重试与黑名单机制
# 第5章 Spark Shuffle解析
## 5.1 Shuffle的核心要点
### 5.1.1 ShuffleMapStage与ResultStage
## 5.2 HashShuffle解析
### 5.2.1 未优化的HashShuffle
### 5.2.2 优化后的HashShuffle
## 5.3 SortShuffle解析
### 5.3.1 普通SortShuffle
### 5.3.2 bypass SortShuffle
# 第6章 Spark内存管理
## 6.1 堆内和堆外内存规划
## 6.2 内存空间分配
## 6.3 存储内存管理
## 6.4 执行内存管理
