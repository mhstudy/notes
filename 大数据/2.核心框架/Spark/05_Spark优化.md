# Spark 优化

> 🔗 **官方文档**：https://spark.apache.org/docs/latest/tuning.html
> 📌 **学习版本**：Spark 3.x

---

## 第1章 常规性能调优 🔥

### 1.1 最优资源配置 🔥🔥

```bash
spark-submit \
    --master yarn \
    --deploy-mode cluster \
    --num-executors 50 \         # Executor 数量
    --executor-cores 4 \         # 每个 Executor 核数
    --executor-memory 8g \       # 每个 Executor 内存
    --driver-memory 4g \         # Driver 内存
    --conf spark.default.parallelism=200 \  # 默认并行度
    --class com.atguigu.spark.App app.jar
```

> 💡 经验公式：
> - Executor 数 = 集群总 CPU 核数 / 每 Executor 核数
> - 并行度 = Executor 数 × 每 Executor 核数 × (2~3)

### 1.2 RDD 优化 🔥

| 优化点 | 说明 |
|:---|:---|
| **RDD 复用** | 对多次使用的 RDD 进行 cache/persist |
| **避免创建重复 RDD** | 一个数据集只创建一个 RDD |
| **尽早 filter** | 减少后续操作数据量 |

### 1.3 并行度调节 ⭐

```scala
// 方式一：配置参数
spark.default.parallelism = 200

// 方式二：代码设置
rdd.repartition(200)
sc.textFile("path", minPartitions = 200)
```

> 💡 并行度建议设置为 **CPU 总核数的 2~3 倍**。

### 1.4 广播大变量 🔥

```scala
// 不使用广播: 每个 Task 都会复制一份变量（内存浪费）
val list = List(...)
rdd.filter(x => list.contains(x))

// 使用广播: 每个 Executor 只存一份
val bcList = sc.broadcast(list)
rdd.filter(x => bcList.value.contains(x))
```

### 1.5 Kryo 序列化 ⭐

```scala
// Kryo 比 Java 序列化快 10 倍，体积小 2~5 倍
conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
conf.registerKryoClasses(Array(classOf[MyClass]))
```

### 1.6 调节本地化等待时长 📝

```scala
spark.locality.wait = 3s  // 默认 3 秒
// 适当增大可提高本地化比例，但会增加等待时间
```

---

## 第2章 算子调优 🔥

### 2.1 mapPartitions 替代 map 🔥

```scala
// map: 每条数据调用一次函数
rdd.map(x => func(x))

// mapPartitions: 每个分区调用一次函数（减少函数调用开销）
rdd.mapPartitions { iter =>
    val conn = getConnection()  // 每分区创建一次连接
    val result = iter.map(x => process(conn, x))
    conn.close()
    result
}
```

> ⚠️ `mapPartitions` 一次性加载整个分区数据到内存，数据量大时可能 OOM。

### 2.2 foreachPartition 优化数据库操作 🔥

```scala
// 错误: 每条数据创建一个连接
rdd.foreach { record =>
    val conn = DriverManager.getConnection(url)  // 频繁创建！
    // insert ...
    conn.close()
}

// 正确: 每个分区创建一个连接
rdd.foreachPartition { iter =>
    val conn = DriverManager.getConnection(url)
    iter.foreach { record =>
        // batch insert ...
    }
    conn.close()
}
```

### 2.3 reduceByKey 替代 groupByKey 🔥🔥

```scala
// groupByKey: 所有数据 Shuffle，再聚合（数据量大）
rdd.groupByKey().mapValues(_.sum)

// reduceByKey: Map 端预聚合 + Shuffle + Reduce 端聚合（数据量小）
rdd.reduceByKey(_ + _)
```

> 💡 `reduceByKey` 自带 Map 端 **Combiner**，减少 Shuffle 数据量！

### 2.4 filter + coalesce 🔥

```scala
// filter 后分区数据不均匀，用 coalesce 缩减分区
rdd.filter(condition)
   .coalesce(numPartitions)  // 不产生 Shuffle
```

---

## 第3章 Shuffle 调优 🔥🔥

### 3.1 核心参数

| 参数 | 默认值 | 说明 |
|:---|:---|:---|
| `spark.shuffle.file.buffer` | **32k** | Map 端写磁盘缓冲区（建议 64k） |
| `spark.reducer.maxSizeInFlight` | **48m** | Reduce 端拉取数据缓冲区（建议 96m） |
| `spark.shuffle.io.maxRetries` | **3** | 拉取数据失败重试次数（建议 6） |
| `spark.shuffle.io.retryWait` | **5s** | 重试等待间隔（建议 60s） |
| `spark.shuffle.sort.bypassMergeThreshold` | **200** | Bypass 阈值（分区数 ≤ 该值跳过排序） |
| `spark.shuffle.compress` | true | Shuffle 数据是否压缩 |

```scala
conf.set("spark.shuffle.file.buffer", "64k")
conf.set("spark.reducer.maxSizeInFlight", "96m")
conf.set("spark.shuffle.io.maxRetries", "6")
conf.set("spark.shuffle.io.retryWait", "60s")
```

---

## 第4章 JVM 调优 ⭐

### 4.1 调节 Executor 堆外内存

```bash
--conf spark.executor.memoryOverhead=2g  # 堆外内存（默认 max(384MB, 0.1*executorMemory)）
```

> 当出现 `Executor lost` 或 `OOM` 时，适当增大堆外内存。

### 4.2 GC 优化

```bash
--conf spark.executor.extraJavaOptions="-XX:+UseG1GC -XX:InitiatingHeapOccupancyPercent=35"
```

---

## 第5章 数据倾斜 🔥🔥🔥

### 5.1 如何定位数据倾斜

```
现象:
1. 绝大多数 Task 很快完成，个别 Task 运行很慢（甚至 OOM）
2. Spark UI 查看 Stage 详情，某些 Task 处理数据量远大于其他

定位方法:
1. 看 Spark UI 的 Task Metrics
2. 对 key 采样统计: rdd.sample(false, 0.1).countByKey()
3. 看日志中的 Shuffle Read 数据量
```

### 5.2 解决方案 🔥🔥

#### 方案一：聚合原数据（源头解决）

在 Hive/数据源层面预聚合，减少 Spark 处理的数据粒度。

#### 方案二：过滤倾斜 Key

```scala
// 如果倾斜 Key 是无效数据（如 null、空值），直接过滤
rdd.filter(_._1 != null && _._1 != "")
```

#### 方案三：提高 Shuffle 并行度

```scala
rdd.reduceByKey(_ + _, numPartitions = 1000)
```

#### 方案四：随机前缀 + 二次聚合（聚合类倾斜最佳方案）🔥🔥

```scala
// 第一次聚合：加随机前缀打散
val prefixRDD = rdd.map { case (key, value) =>
    val prefix = scala.util.Random.nextInt(10)  // 0~9 随机前缀
    (s"${prefix}_${key}", value)
}
val firstAgg = prefixRDD.reduceByKey(_ + _)

// 第二次聚合：去掉前缀，再次聚合
val result = firstAgg.map { case (key, value) =>
    (key.split("_")(1), value)
}.reduceByKey(_ + _)
```

#### 方案五：Map Join（Join 类倾斜）🔥

```scala
// 大表 Join 小表：广播小表，避免 Shuffle
val smallDF = spark.read.parquet("small_table")
val largeDF = spark.read.parquet("large_table")

// 自动 Broadcast
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 10485760)  // 10MB
val result = largeDF.join(broadcast(smallDF), "key")
```

#### 方案六：采样 + 单独处理倾斜 Key

```scala
// 1. 采样找出倾斜 Key
val skewedKeys = rdd.sample(false, 0.1)
    .countByKey()
    .filter(_._2 > threshold)
    .keys.toSet

// 2. 分离倾斜数据和正常数据
val skewedRDD = rdd.filter(x => skewedKeys.contains(x._1))
val normalRDD = rdd.filter(x => !skewedKeys.contains(x._1))

// 3. 倾斜数据加随机前缀处理，正常数据正常处理
// 4. 最终 union 结果
```

#### 方案七：随机数扩容 Join

```scala
// 大表 Join 大表，一方倾斜
// 1. 倾斜侧加随机前缀 (0~N)
// 2. 非倾斜侧数据膨胀 N 倍（每条复制 N 份，加前缀 0~N-1）
// 3. 两侧 Join
```

### 5.3 数据倾斜方案总结

| 方案 | 适用场景 | 效果 |
|:---|:---|:---|
| 过滤倾斜 Key | Key 为无效值 | ⭐⭐⭐ |
| 提高并行度 | 轻度倾斜 | ⭐ |
| 随机前缀+二次聚合 | **聚合类倾斜** | ⭐⭐⭐⭐ |
| Broadcast Join | **大小表 Join** | ⭐⭐⭐⭐⭐ |
| 采样+单独处理 | 少量 Key 倾斜 | ⭐⭐⭐ |
| 随机数扩容 Join | 大大表 Join | ⭐⭐ |

---

## 第6章 Spark 3.x 新特性 🔥

### 6.1 AQE（Adaptive Query Execution）🔥🔥

```scala
// Spark 3.0 默认开启
spark.sql.adaptive.enabled = true

AQE 自动优化:
1. 自动合并小分区 (Coalesce Shuffle Partitions)
2. 自动转换 Sort Merge Join → Broadcast Hash Join
3. 自动优化数据倾斜的 Join (Skew Join Optimization)
```

### 6.2 DPP（Dynamic Partition Pruning）

```sql
-- 动态分区裁剪：根据 Join 条件自动裁剪不需要的分区
-- Spark 3.0 自动支持
SELECT * FROM fact_table f
JOIN dim_table d ON f.date_key = d.date_key
WHERE d.year = 2024
-- DPP 会自动将 year=2024 的过滤条件下推到 fact_table 的分区裁剪中
```

---

## 🔥 面试高频题

### Q1：Spark 数据倾斜如何解决？
> 视场景选择：聚合类用「随机前缀+二次聚合」；大小表 Join 用「Broadcast Join」；少量 Key 倾斜用「采样+单独处理」；通用方案提高并行度。Spark 3.x 的 AQE 能自动处理部分倾斜。

### Q2：reduceByKey 和 groupByKey 的区别？
> `reduceByKey` 在 Map 端预聚合（自带 Combiner），减少 Shuffle 数据量，性能更好。`groupByKey` 不预聚合，所有数据 Shuffle 到 Reduce 端再聚合。**能用 reduceByKey 就不用 groupByKey**。

### Q3：Spark 的 Shuffle 如何调优？
> 增大 Map 端缓冲区（64k）、Reduce 端拉取缓冲区（96m）、增大重试次数和间隔、使用 Bypass SortShuffle（分区数 ≤ 200 且无预聚合）、开启 Shuffle 压缩。

### Q4：Spark 3.x AQE 有哪些优化？
> 自动合并小分区、自动将 Sort Merge Join 转为 Broadcast Hash Join、自动优化倾斜 Join。
