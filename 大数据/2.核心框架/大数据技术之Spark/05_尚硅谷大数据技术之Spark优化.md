# 第1章 Spark 性能调优
## 1.1 常规性能调优
### 1.1.1 常规性能调优一：最优资源配置
### 1.1.2 常规性能调优二：RDD优化
### 1.1.3 常规性能调优三：并行度调节
### 1.1.4 常规性能调优四：广播大变量
### 1.1.5 常规性能调优五：Kryo序列化
### 1.1.6 常规性能调优六：调节本地化等待时长
## 1.2 算子调优
### 1.2.1 算子调优一：mapPartitions
### 1.2.2 算子调优二：foreachPartition优化数据库操作
### 1.2.3 算子调优三：filter与coalesce的配合使用
### 1.2.4 算子调优四：repartition解决SparkSQL低并行度问题
### 1.2.5 算子调优五：reduceByKey预聚合
## 1.3 Shuffle调优
### 1.3.1 Shuffle调优一：调节map端缓冲区大小
### 1.3.2 Shuffle调优二：调节reduce端拉取数据缓冲区大小
### 1.3.3 Shuffle调优三：调节reduce端拉取数据重试次数
### 1.3.4 Shuffle调优四：调节reduce端拉取数据等待间隔
### 1.3.5 Shuffle调优五：调节SortShuffle排序操作阈值
## 1.4 JVM调优
### 1.4.1 JVM调优一：降低cache操作的内存占比
### 1.4.2 JVM调优二：调节Executor堆外内存
### 1.4.3 JVM调优三：调节连接等待时长
# 第2章 Spark数据倾斜
## 2.1 解决方案一：聚合原数据
## 2.2 解决方案二：过滤导致倾斜的key
## 2.3 解决方案三：提高shuffle操作中的reduce并行度
## 2.4 解决方案四：使用随机key实现双重聚合
## 2.5 解决方案五：将reduce join转换为map join
## 2.6 解决方案六：sample采样对倾斜key单独进行join
## 2.7 解决方案七：使用随机数扩容进行join
# 第3章 Spark故障排除
## 3.1 故障排除一：控制reduce端缓冲大小以避免OOM
## 3.2 故障排除二：JVM GC导致的shuffle文件拉取失败
## 3.3 故障排除三：解决各种序列化导致的报错
## 3.4 故障排除四：解决算子函数返回NULL导致的问题
## 3.5 故障排除五：解决YARN-CLIENT模式导致的网卡流量激增问题
## 3.6 故障排除六：解决YARN-CLUSTER模式的JVM栈内存溢出无法执行问题
## 3.7 故障排除七：解决SparkSQL导致的JVM栈内存溢出
## 3.8 故障排除八：持久化与checkpoint的使用
