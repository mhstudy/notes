# Scala

> 🔗 **官方网站**：https://www.scala-lang.org/
> 📖 **官方文档**：https://docs.scala-lang.org/
> 📌 **学习版本**：Scala 2.12.x（Spark 默认版本）

---

## 第1章 Scala 概述 ⭐

### 1.1 什么是 Scala

Scala 是一门**多范式编程语言**，融合了面向对象和函数式编程，运行在 JVM 上，与 Java 无缝互操作。

**大数据中的重要性**：Spark、Kafka、Flink 都是 Scala 编写的。

---

## 第2章 基础语法 ⭐

### 2.1 变量与类型

```scala
// val：不可变（推荐）  var：可变
val name: String = "张三"
var age: Int = 25
age = 26  // OK

// 类型推断
val x = 10         // Int
val y = 3.14       // Double
val z = "hello"    // String

// Scala 与 Java 类型对应
// Int → int/Integer
// Long → long/Long
// Double → double/Double
// Boolean → boolean/Boolean
// String → String（共用 java.lang.String）
// Unit → void
// Nothing → 无返回值（异常）
// Any → 所有类型的父类
// Null → 引用类型的空值
```

### 2.2 函数与方法 🔥

```scala
// 方法定义
def add(a: Int, b: Int): Int = {
    a + b
}

// 简写
def add(a: Int, b: Int): Int = a + b

// 无返回值
def greet(name: String): Unit = println(s"Hello, $name")

// 默认参数
def power(base: Int, exp: Int = 2): Int = Math.pow(base, exp).toInt

// 可变参数
def sum(nums: Int*): Int = nums.sum

// 匿名函数（Lambda）🔥
val double = (x: Int) => x * 2
val add = (a: Int, b: Int) => a + b
```

### 2.3 字符串

```scala
// 字符串插值
val name = "Scala"
println(s"Hello, $name")              // s 插值
println(s"1+1=${1+1}")
println(f"PI = ${Math.PI}%.2f")       // f 格式化

// 多行字符串
val sql = """
    |SELECT *
    |FROM users
    |WHERE age > 18
""".stripMargin
```

---

## 第3章 集合操作 🔥🔥

> 集合操作是 Spark 编程的基础！

### 3.1 List

```scala
val list = List(1, 2, 3, 4, 5)

// 常用操作
list.head          // 1
list.tail          // List(2, 3, 4, 5)
list.isEmpty       // false
list.length        // 5
list.reverse       // List(5, 4, 3, 2, 1)
list.take(3)       // List(1, 2, 3)
list.drop(2)       // List(3, 4, 5)

// 追加
0 :: list          // List(0, 1, 2, 3, 4, 5)   头部追加
list :+ 6          // List(1, 2, 3, 4, 5, 6)   尾部追加
list ++ List(6,7)  // 拼接
```

### 3.2 高阶函数（核心！）🔥🔥

```scala
val list = List(1, 2, 3, 4, 5, 6)

// map：一对一转换
list.map(_ * 2)              // List(2, 4, 6, 8, 10, 12)
list.map(x => x * x)        // List(1, 4, 9, 16, 25, 36)

// filter：过滤
list.filter(_ > 3)           // List(4, 5, 6)
list.filter(_ % 2 == 0)     // List(2, 4, 6)

// flatMap：一对多 + 扁平化
val words = List("hello world", "hi scala")
words.flatMap(_.split(" "))  // List(hello, world, hi, scala)

// reduce / fold
list.reduce(_ + _)           // 21（求和）
list.fold(0)(_ + _)          // 21（带初始值的求和）
list.foldLeft(0)(_ + _)     // 从左折叠

// groupBy：分组
val nums = List(1, 2, 3, 4, 5, 6)
nums.groupBy(_ % 2 == 0)    // Map(false->List(1,3,5), true->List(2,4,6))

// sortBy / sorted
list.sorted                  // 升序
list.sortBy(-_)              // 降序
list.sortWith(_ > _)         // 降序

// zip
val names = List("a", "b", "c")
val scores = List(90, 80, 70)
names.zip(scores)            // List((a,90), (b,80), (c,70))

// distinct / count
list.distinct
list.count(_ > 3)            // 3
```

### 3.3 WordCount 示例（Spark 核心思想）🔥

```scala
val lines = List("hello world", "hello scala", "hello spark")

val result = lines
    .flatMap(_.split(" "))       // 扁平化：["hello","world","hello","scala",...]
    .map((_, 1))                  // 映射：[("hello",1),("world",1),...]
    .groupBy(_._1)                // 分组：Map("hello"->List(("hello",1),...), ...)
    .map(t => (t._1, t._2.size)) // 聚合：Map("hello"->3, "world"->1, ...)
    .toList
    .sortBy(-_._2)                // 降序排序

println(result)
// List((hello,3), (world,1), (scala,1), (spark,1))
```

### 3.4 Map

```scala
val map = Map("a" -> 1, "b" -> 2, "c" -> 3)
map("a")                     // 1
map.getOrElse("d", 0)       // 0
map.keys                     // Set(a, b, c)
map.values                   // Iterable(1, 2, 3)
map.map(t => (t._1, t._2 * 10))  // Map(a->10, b->20, c->30)
```

### 3.5 Tuple（元组）

```scala
val t2 = ("hello", 100)
val t3 = ("hello", 100, true)
t2._1   // "hello"
t2._2   // 100
```

---

## 第4章 模式匹配 🔥

```scala
// 基本匹配
val x: Any = 100
x match {
    case 1          => "one"
    case s: String  => s"String: $s"
    case i: Int if i > 0 => s"正整数: $i"
    case _          => "other"    // 默认
}

// 匹配集合
val list = List(1, 2, 3)
list match {
    case List(1, _, _) => "以1开头的三元素列表"
    case List(1, _*)   => "以1开头的列表"
    case _             => "其他"
}

// 样例类匹配
case class Person(name: String, age: Int)
val p = Person("张三", 25)
p match {
    case Person("张三", age) => s"张三 $age 岁"
    case Person(name, age) if age > 18 => s"$name 成年"
    case _ => "未知"
}
```

---

## 第5章 隐式转换 ⭐

```scala
// 隐式转换函数
implicit def intToString(x: Int): String = x.toString

val s: String = 123  // 自动调用 intToString

// 隐式参数
def greet(name: String)(implicit greeting: String): Unit = {
    println(s"$greeting, $name!")
}
implicit val defaultGreeting: String = "Hello"
greet("Scala")  // Hello, Scala!

// 隐式类（扩展方法）
implicit class RichString(val str: String) {
    def isEmail: Boolean = str.contains("@")
}
"test@qq.com".isEmail  // true
```

---

## 第6章 面试要点 🔥

### Q1：val 和 var 的区别？
> `val` 不可变（类似 Java final），`var` 可变。推荐使用 `val`。

### Q2：map、flatMap、filter 的区别？
> `map`：一对一转换。`flatMap`：一对多转换并扁平化。`filter`：过滤。

### Q3：Scala 在大数据中的作用？
> Spark、Kafka、Flink 底层都是 Scala 编写。学习 Scala 的集合操作是理解 Spark RDD/DataFrame API 的基础。
