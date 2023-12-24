# 第1章 Yarn资源调度器
## 1.1 Yarn基础架构
## 1.2 Yarn工作机制
## 1.3 作业提交全过程
## 1.4 Yarn调度器和调度算法
### 1.4.1 先进先出调度器（FIFO）
### 1.4.2 容量调度器（Capacity Scheduler）
### 1.4.3 公平调度器（Fair Scheduler）
## 1.5 Yarn常用命令
### 1.5.1 yarn application查看任务
### 1.5.2 yarn logs查看日志
### 1.5.3 yarn applicationattempt查看尝试运行的任务
### 1.5.4 yarn container查看容器
### 1.5.5 yarn node查看节点状态
### 1.5.6 yarn rmadmin更新配置
### 1.5.7 yarn queue查看队列
## 1.6 Yarn生产环境核心参数
# 第2章 Yarn案例实操
## 2.1 Yarn生产环境核心参数配置案例
## 2.2 容量调度器多队列提交案例
### 2.2.1 需求
### 2.2.2 配置多队列的容量调度器
### 2.2.3 向Hive队列提交任务
### 2.2.4 任务优先级
## 2.3 公平调度器案例
### 2.3.1 需求
### 2.3.2 配置多队列的公平调度器
### 2.3.3 测试提交任务
## 2.4 Yarn的Tool接口案例
