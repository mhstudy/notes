# Spark Core

> 🔗 **官方网站**：https://spark.apache.org/
> 📖 **官方文档**：https://spark.apache.org/docs/latest/
> 📌 **学习版本**：Spark 3.x

---

## 第1章 Spark 概述 ⭐

### 1.1 什么是 Spark

Spark 是一个快速、通用的**大数据计算引擎**，比 MapReduce 快 **10~100 倍**（内存计算）。

### 1.2 Spark vs MapReduce 🔥

| 对比 | Spark | MapReduce |
|:---|:---|:---|
| 速度 | **内存计算，快 10~100 倍** | 磁盘 IO，慢 |
| 编程模型 | RDD / DataFrame / DataSet | Map + Reduce |
| 语言 | Scala / Java / Python / R | Java |
| DAG | 支持，优化执行计划 | 线性 MR |
| 迭代计算 | 支持（ML 场景） | 不擅长 |

### 1.3 Spark 架构 🔥

```
┌─────────────┐
│   Driver    │  ← 创建 SparkContext，划分 Stage，调度 Task
└──────┬──────┘
       │
  ┌────┼────┬────┐
  ▼    ▼    ▼    ▼
┌────┐┌────┐┌────┐┌────┐
│Exec││Exec││Exec││Exec│  ← Executor：执行 Task，缓存数据
└────┘└────┘└────┘└────┘
```

| 组件 | 说明 |
|:---|:---|
| **Driver** | 运行 main 方法，创建 SparkContext，划分 Job/Stage/Task |
| **Executor** | 在 Worker 节点运行，执行 Task，管理内存和磁盘存储 |
| **Cluster Manager** | 资源管理（Standalone / YARN / Mesos / K8s） |

---

## 第2章 RDD 核心 🔥🔥

### 2.1 什么是 RDD

RDD（Resilient Distributed Dataset）弹性分布式数据集，Spark 最核心的抽象。

**五大属性**（面试常问）：
1. **分区列表** — 数据被划分为多个分区
2. **计算函数** — 每个分区有计算函数
3. **依赖关系** — RDD 之间的血缘关系（Lineage）
4. **分区器** — Key-Value RDD 的分区方式
5. **优先位置** — 数据本地性

### 2.2 RDD 创建

```scala
val sc = new SparkContext(conf)

// 从集合创建
val rdd1 = sc.parallelize(List(1, 2, 3, 4, 5))
val rdd2 = sc.makeRDD(List(1, 2, 3, 4, 5))

// 从文件创建
val rdd3 = sc.textFile("hdfs://hadoop102:8020/input/words.txt")
val rdd4 = sc.textFile("input/words.txt", 3)  // 指定最小分区数
```

### 2.3 Transformation 算子 🔥🔥

```scala
val rdd = sc.makeRDD(List(1, 2, 3, 4, 5, 6))

// Value 类型
rdd.map(_ * 2)                    // [2,4,6,8,10,12]
rdd.flatMap(x => List(x, x*10))  // [1,10,2,20,3,30,...]
rdd.filter(_ > 3)                 // [4,5,6]
rdd.distinct()                    // 去重
rdd.sortBy(x => x, ascending=false) // 降序排序
rdd.sample(false, 0.5)           // 采样

// 双 Value 类型
val rdd1 = sc.makeRDD(List(1,2,3))
val rdd2 = sc.makeRDD(List(3,4,5))
rdd1.union(rdd2)                  // 并集 [1,2,3,3,4,5]
rdd1.intersection(rdd2)          // 交集 [3]
rdd1.subtract(rdd2)              // 差集 [1,2]
rdd1.zip(rdd2)                   // 拉链 [(1,3),(2,4),(3,5)]

// Key-Value 类型 🔥
val pairRdd = sc.makeRDD(List(("a",1),("b",2),("a",3)))
pairRdd.groupByKey()              // [("a",[1,3]),("b",[2])]
pairRdd.reduceByKey(_ + _)        // [("a",4),("b",2)] 🔥 推荐！
pairRdd.aggregateByKey(0)(math.max(_, _), _ + _)  // 分区内取最大，分区间求和
pairRdd.sortByKey()               // 按 key 排序

// join
val rdd3 = sc.makeRDD(List(("a",10),("b",20)))
pairRdd.join(rdd3)                // [("a",(1,10)),("a",(3,10)),("b",(2,20))]
pairRdd.cogroup(rdd3)             // [("a",([1,3],[10])),("b",([2],[20]))]
```

> 💡 **面试必问**：`reduceByKey` vs `groupByKey` 的区别？
> `reduceByKey` 会在 Map 端预聚合（Combine），减少 Shuffle 数据量，**性能更好**！
> `groupByKey` 不预聚合，全部 Shuffle 到 Reduce 端。

### 2.4 Action 算子

```scala
rdd.collect()        // 返回数组
rdd.count()          // 计数
rdd.first()          // 第一个元素
rdd.take(3)          // 前3个
rdd.reduce(_ + _)    // 聚合
rdd.foreach(println) // 遍历
rdd.saveAsTextFile("output/")  // 保存到文件
rdd.countByKey()     // 按key计数
```

### 2.5 WordCount 完整案例 🔥

```scala
import org.apache.spark.{SparkConf, SparkContext}

object WordCount {
    def main(args: Array[String]): Unit = {
        val conf = new SparkConf().setAppName("WordCount").setMaster("local[*]")
        val sc = new SparkContext(conf)

        sc.textFile("input/words.txt")
          .flatMap(_.split(" "))
          .map((_, 1))
          .reduceByKey(_ + _)
          .sortBy(_._2, ascending = false)
          .collect()
          .foreach(println)

        sc.stop()
    }
}
```

---

## 第3章 依赖与 Shuffle 🔥🔥

### 3.1 窄依赖 vs 宽依赖 🔥

| 类型 | 说明 | 示例 |
|:---|:---|:---|
| **窄依赖** | 父 RDD 一个分区→子 RDD 一个分区 | map, filter, union |
| **宽依赖** | 父 RDD 一个分区→子 RDD 多个分区（**Shuffle**） | groupByKey, reduceByKey, join |

### 3.2 Stage 划分

- 遇到**宽依赖**就划分一个新的 Stage
- Stage 内部全是窄依赖（可以 pipeline 执行）

```
Job → Stage 1 (map, filter) → Shuffle → Stage 2 (reduceByKey) → Action
```

### 3.3 持久化与缓存 🔥

```scala
// 缓存
rdd.cache()                          // 默认 MEMORY_ONLY
rdd.persist(StorageLevel.MEMORY_AND_DISK)  // 内存+磁盘

// Checkpoint（可靠，写 HDFS）
sc.setCheckpointDir("hdfs://hadoop102:8020/checkpoint")
rdd.checkpoint()
```

| 方式 | 存储位置 | 可靠性 | 血缘关系 |
|:---|:---|:---|:---|
| cache | 内存/磁盘 | 低 | 保留 |
| checkpoint | **HDFS** | **高** | 切断 |

---

## 第4章 面试题 🔥🔥🔥

### Q1：Spark 的执行流程？
> Driver 创建 SparkContext → 划分 Job（遇到 Action）→ 划分 Stage（遇到宽依赖）→ 划分 Task（一个分区一个 Task）→ TaskScheduler 发送 Task 到 Executor 执行

### Q2：reduceByKey 和 groupByKey 的区别？
> reduceByKey 在 Map 端预聚合，性能更好。groupByKey 不预聚合，全量 Shuffle。

### Q3：cache 和 checkpoint 的区别？
> cache 存内存/磁盘，不可靠，保留血缘。checkpoint 存 HDFS，可靠，切断血缘。

### Q4：Spark 如何处理数据倾斜？
> 1. 增大并行度 `repartition`
> 2. 两阶段聚合（加随机前缀）
> 3. 广播小表（BroadcastJoin）
> 4. 过滤异常 key
> 5. 自定义 Partitioner
