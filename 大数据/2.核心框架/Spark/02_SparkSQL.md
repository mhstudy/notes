# Spark SQL

> 🔗 **官方文档**：https://spark.apache.org/docs/latest/sql-programming-guide.html
> 📌 **学习版本**：Spark 3.x

---

## 第1章 Spark SQL 概述 ⭐

### 1.1 什么是 Spark SQL

Spark SQL 是 Spark 的**结构化数据处理模块**，提供 DataFrame 和 DataSet API，支持 SQL 查询。

### 1.2 DataFrame vs DataSet vs RDD 🔥

| 对比 | RDD | DataFrame | DataSet |
|:---|:---|:---|:---|
| 类型安全 | ✅ | ❌ | ✅ |
| 优化 | 无 | Catalyst 优化 | Catalyst 优化 |
| Schema | 无 | 有 | 有 |
| 序列化 | Java 序列化 | **Tungsten** 编码 | **Tungsten** 编码 |
| API | 函数式 | SQL + 函数式 | SQL + 函数式 |

---

## 第2章 DataFrame 操作 🔥

```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession.builder()
    .appName("SparkSQL")
    .master("local[*]")
    .enableHiveSupport()
    .getOrCreate()

import spark.implicits._

// 创建 DataFrame
val df = spark.read.json("input/user.json")
df.show()
df.printSchema()

// DSL 语法
df.select("name", "age").show()
df.select($"name", $"age" + 1).show()
df.filter($"age" > 20).show()
df.groupBy("dept").count().show()
df.orderBy($"age".desc).show()

// SQL 语法
df.createOrReplaceTempView("user")
spark.sql("SELECT name, age FROM user WHERE age > 20").show()
spark.sql("""
    SELECT dept, COUNT(*) cnt, AVG(age) avg_age
    FROM user
    GROUP BY dept
    HAVING cnt > 2
    ORDER BY avg_age DESC
""").show()
```

---

## 第3章 DataSet 操作 ⭐

### 3.1 DataFrame 转 DataSet

```scala
// 定义样例类
case class User(name: String, age: Long)

// DataFrame → DataSet
val userDS: Dataset[User] = df.as[User]

// 利用类型安全操作
userDS.filter(_.age > 20).show()
userDS.map(u => (u.name, u.age * 2)).show()
```

### 3.2 RDD ↔ DataFrame ↔ DataSet 转换 🔥

```
        .toDF()              .as[T]
  RDD  ────────▶ DataFrame ────────▶ DataSet
  RDD  ◀──────── DataFrame ◀──────── DataSet
        .rdd                  .toDF()
```

```scala
// RDD → DataFrame
val rdd = sc.textFile("input.txt").map(_.split(",")).map(x => (x(0), x(1).toInt))
val df = rdd.toDF("name", "age")

// DataFrame → RDD
val rdd2 = df.rdd  // RDD[Row]

// RDD → DataSet
val ds = rdd.map { case (name, age) => User(name, age) }.toDS()
```

---

## 第4章 数据读写 ⭐

### 4.1 通用读写方式

```scala
// 读取 JSON
val jsonDf = spark.read.json("input/user.json")

// 读取 CSV
val csvDf = spark.read
    .option("header", "true")
    .option("inferSchema", "true")
    .option("sep", ",")
    .csv("input/data.csv")

// 读取 Parquet（默认格式）
val parquetDf = spark.read.parquet("input/data.parquet")

// 读取 ORC
val orcDf = spark.read.orc("input/data.orc")

// 写出
df.write.mode("overwrite").json("output/json")
df.write.mode("append").parquet("output/parquet")
df.write.mode("overwrite")
    .option("header", "true")
    .csv("output/csv")
```

### 4.2 与 JDBC 交互

```scala
// 读取 MySQL
val jdbcDf = spark.read.format("jdbc")
    .option("url", "jdbc:mysql://hadoop102:3306/gmall")
    .option("dbtable", "user_info")
    .option("user", "root")
    .option("password", "000000")
    .load()

// 写入 MySQL
df.write.format("jdbc")
    .option("url", "jdbc:mysql://hadoop102:3306/gmall")
    .option("dbtable", "user_result")
    .option("user", "root")
    .option("password", "000000")
    .mode("overwrite")
    .save()
```

---

## 第5章 与 Hive 集成 🔥

```scala
// 读取 Hive 表
val hiveDf = spark.sql("SELECT * FROM gmall.dwd_user_info WHERE dt = '2023-06-15'")

// 写入 Hive 表
df.write.mode("overwrite").saveAsTable("gmall.ads_user_stats")

// 动态分区写入
spark.conf.set("hive.exec.dynamic.partition", "true")
spark.conf.set("hive.exec.dynamic.partition.mode", "nonstrict")
df.write.mode("overwrite").partitionBy("dt").saveAsTable("gmall.dwd_order_info")
```

---

## 第6章 常用函数 ⭐

```scala
import org.apache.spark.sql.functions._

// 窗口函数
df.withColumn("rn", row_number().over(Window.partitionBy("dept").orderBy($"age".desc)))

// 聚合
df.groupBy("dept").agg(
    count("*").as("cnt"),
    avg("age").as("avg_age"),
    max("salary").as("max_salary")
)

// 日期
df.select(current_date(), date_format($"ts", "yyyy-MM-dd"))

// JSON
df.select(get_json_object($"json_col", "$.name"))

// 字符串
df.select(concat_ws("-", $"year", $"month", $"day").as("date_str"))
df.select(regexp_extract($"url", "(\\w+)://([^/]+)", 2).as("domain"))

// 条件表达式
df.withColumn("level", when($"age" < 18, "少年")
    .when($"age" < 30, "青年")
    .otherwise("中年"))
```

---

## 第7章 UDF 自定义函数 🔥

### 7.1 UDF（User Defined Function）

```scala
// 方式1：匿名函数注册
spark.udf.register("toUpper", (s: String) => s.toUpperCase)
spark.sql("SELECT toUpper(name) FROM user").show()

// 方式2：DSL 中使用
import org.apache.spark.sql.functions.udf
val addPrefix = udf((name: String) => s"用户_$name")
df.select(addPrefix($"name").as("new_name")).show()
```

### 7.2 UDAF（User Defined Aggregate Function）

```scala
// Spark 3.x 推荐 Aggregator 方式
import org.apache.spark.sql.{Encoder, Encoders}
import org.apache.spark.sql.expressions.Aggregator

case class Average(var sum: Long, var count: Long)

object MyAvg extends Aggregator[Long, Average, Double] {
    def zero: Average = Average(0L, 0L)
    def reduce(buf: Average, input: Long): Average = {
        buf.sum += input; buf.count += 1; buf
    }
    def merge(b1: Average, b2: Average): Average = {
        b1.sum += b2.sum; b1.count += b2.count; b1
    }
    def finish(buf: Average): Double = buf.sum.toDouble / buf.count
    def bufferEncoder: Encoder[Average] = Encoders.product
    def outputEncoder: Encoder[Double] = Encoders.scalaDouble
}

// 注册使用
spark.udf.register("my_avg", functions.udaf(MyAvg))
spark.sql("SELECT dept, my_avg(salary) FROM emp GROUP BY dept").show()
```

---

## 第8章 Catalyst 优化器 🔥🔥

### 8.1 执行流程

```
SQL / DSL
    ↓  解析（Parser）
Unresolved Logical Plan（未解析逻辑计划）
    ↓  分析（Analyzer）—— 绑定表名、列名
Logical Plan（逻辑计划）
    ↓  优化（Optimizer）—— 谓词下推、列裁剪、常量折叠等
Optimized Logical Plan（优化后逻辑计划）
    ↓  物理规划（Planner）—— 选择 Join 策略等
Physical Plan（物理计划）
    ↓  代码生成（Whole-Stage CodeGen）
RDD 执行
```

### 8.2 关键优化规则

| 优化规则 | 说明 |
|:---|:---|
| **谓词下推（Predicate Pushdown）** 🔥 | WHERE 条件尽早过滤，减少数据量 |
| **列裁剪（Column Pruning）** | 只读取需要的列 |
| **常量折叠（Constant Folding）** | 编译时计算常量表达式 |
| **Join 重排序** | 小表放前面，优化 Join 性能 |
| **分区裁剪（Partition Pruning）** | 跳过不需要的分区 |

---

## 第9章 AQE 自适应查询 🔥🔥（Spark 3.x）

### 9.1 什么是 AQE

AQE（Adaptive Query Execution）是 Spark 3.0 引入的**运行时自适应优化**，在 Shuffle 后根据实际数据统计信息动态调整执行计划。

```scala
// 开启 AQE（Spark 3.2+ 默认开启）
spark.conf.set("spark.sql.adaptive.enabled", "true")
```

### 9.2 三大核心特性

| 特性 | 说明 |
|:---|:---|
| **动态合并 Shuffle 分区** 🔥 | 自动合并过小的分区，避免小任务过多 |
| **动态切换 Join 策略** 🔥 | 运行时发现小表，自动切换为 BroadcastJoin |
| **动态优化倾斜 Join** | 检测到数据倾斜，自动拆分大分区 |

```scala
// 关键配置
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")        // 合并小分区
spark.conf.set("spark.sql.adaptive.coalescePartitions.minPartitionSize", "1m") // 最小分区大小
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")                  // 倾斜 Join 优化
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")       // 倾斜因子
```

---

## 第10章 面试题 🔥🔥

### Q1：DataFrame 和 RDD 的区别？
> DataFrame 有 Schema 信息，经过 Catalyst 优化器优化，性能更好。RDD 无 Schema，无优化。

### Q2：Spark SQL 的执行流程？
> SQL/DSL → 解析为逻辑计划 → Catalyst 优化 → 生成物理计划 → Whole-Stage CodeGen → 生成 RDD 执行

### Q3：谓词下推是什么？
> 将 WHERE 过滤条件尽可能下推到数据源端执行（如 Parquet、ORC、JDBC），减少读取的数据量，大幅提升性能。

### Q4：AQE 的三大特性是什么？
> 动态合并 Shuffle 分区、动态切换 Join 策略（小表自动转 BroadcastJoin）、动态优化数据倾斜 Join。

### Q5：Spark SQL 支持哪些 Join 策略？
> | Join 策略 | 适用场景 |
> |:---|:---|
> | **Broadcast Hash Join** 🔥 | 小表 < 10MB（默认），最快 |
> | **Sort Merge Join** | 大表 Join 大表，默认策略 |
> | **Shuffle Hash Join** | 一侧较小，但超过广播阈值 |
> | **Broadcast Nested Loop Join** | 非等值 Join + 小表 |

### Q6：UDF 和内置函数哪个性能好？
> 内置函数性能更好，因为经过 Catalyst 优化和 Whole-Stage CodeGen。UDF 是黑盒，无法被优化器优化，尽量优先使用内置函数。
