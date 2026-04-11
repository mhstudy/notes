# Java SE 基础

> 📌 **重点**：大数据开发中常用的 Java 核心知识

---

## 第1章 集合框架 🔥🔥

### 1.1 集合体系

```
Collection（接口）
├── List（有序可重复）
│   ├── ArrayList     🔥 数组实现，查询快
│   ├── LinkedList    链表实现，增删快
│   └── Vector        线程安全（过时）
├── Set（无序不重复）
│   ├── HashSet       🔥 HashMap 实现
│   ├── LinkedHashSet 有序
│   └── TreeSet       有序
└── Queue
    └── LinkedList

Map（接口）
├── HashMap          🔥🔥 数组+链表+红黑树(JDK8)
├── LinkedHashMap    有序
├── TreeMap          排序
├── Hashtable        线程安全（过时）
└── ConcurrentHashMap 🔥 线程安全（分段锁/CAS）
```

### 1.2 HashMap 原理 🔥🔥

```
JDK 8: 数组 + 链表 + 红黑树
├── 默认初始容量: 16
├── 负载因子: 0.75
├── 链表长度 ≥ 8 且数组长度 ≥ 64 → 转红黑树
├── 红黑树节点 ≤ 6 → 转回链表
└── 扩容: 2倍扩容

put 流程:
1. hash(key) → 计算数组下标
2. 数组位置为空 → 直接放入
3. 有值 → 判断 key 是否相同
   - 相同 → 覆盖 value
   - 不同 → 链表尾插（JDK8）/ 红黑树
4. 判断是否需要扩容
```

---

## 第2章 多线程 🔥

### 2.1 线程创建

```java
// 方式1：继承 Thread
new Thread(() -> System.out.println("线程1")).start();

// 方式2：实现 Runnable
Runnable task = () -> System.out.println("线程2");
new Thread(task).start();

// 方式3：Callable + FutureTask（有返回值）
Callable<Integer> callable = () -> 42;
FutureTask<Integer> future = new FutureTask<>(callable);
new Thread(future).start();
System.out.println(future.get());  // 42

// 方式4：线程池（推荐）🔥
ExecutorService pool = Executors.newFixedThreadPool(3);
pool.submit(() -> System.out.println("池中线程"));
pool.shutdown();
```

### 2.2 线程安全 🔥

```java
// synchronized
public synchronized void method() { /* ... */ }

// Lock
ReentrantLock lock = new ReentrantLock();
lock.lock();
try { /* ... */ } finally { lock.unlock(); }

// volatile：保证可见性，不保证原子性
private volatile boolean flag = false;

// CAS（乐观锁）
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();  // 原子自增
```

---

## 第3章 IO 流 ⭐

```java
// 文件读写
try (BufferedReader br = new BufferedReader(new FileReader("input.txt"));
     BufferedWriter bw = new BufferedWriter(new FileWriter("output.txt"))) {
    String line;
    while ((line = br.readLine()) != null) {
        bw.write(line);
        bw.newLine();
    }
}

// 序列化（Hadoop/Spark 中大量使用）
public class User implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private int age;
}
```

---

## 第4章 JVM 基础 🔥🔥

### 4.1 内存模型

```
┌───────────────────────────────────────────────┐
│              JVM 运行时数据区                    │
├──────────────┬───────────────┬────────────────┤
│ 线程共享      │               │ 线程私有        │
│ ┌──────────┐ │ ┌───────────┐ │ ┌────────────┐ │
│ │   堆     │ │ │  方法区    │ │ │ 虚拟机栈   │ │
│ │ (Heap)   │ │ │(MetaSpace)│ │ │ (Stack)    │ │
│ │ 对象实例  │ │ │ 类信息    │ │ │ 局部变量   │ │
│ │ 数组     │ │ │ 常量池    │ │ │ 操作数栈   │ │
│ └──────────┘ │ └───────────┘ │ │ 方法调用   │ │
│              │               │ └────────────┘ │
│              │               │ ┌────────────┐ │
│              │               │ │ 程序计数器  │ │
│              │               │ └────────────┘ │
│              │               │ ┌────────────┐ │
│              │               │ │ 本地方法栈  │ │
│              │               │ └────────────┘ │
└──────────────┴───────────────┴────────────────┘
```

### 4.2 GC 垃圾回收 🔥

```
堆内存分代:
┌─────────────────────────────────────────┐
│ Young Generation (新生代) 1/3            │
│ ┌──────────┬─────────┬─────────┐        │
│ │  Eden    │   S0    │   S1    │        │
│ │  (8/10)  │ (1/10)  │ (1/10)  │        │
│ └──────────┴─────────┴─────────┘        │
├─────────────────────────────────────────┤
│ Old Generation (老年代) 2/3              │
└─────────────────────────────────────────┘

GC 过程:
1. 对象优先在 Eden 分配
2. Eden 满 → Minor GC → 存活对象复制到 S0/S1
3. 对象年龄达到阈值(默认15) → 晋升老年代
4. 老年代满 → Major GC / Full GC（STW 时间长）
```

### 4.3 常用 JVM 参数

```bash
# Hadoop/Spark/Flink 常用 JVM 配置
-Xms4g                    # 初始堆大小
-Xmx4g                    # 最大堆大小（建议与 -Xms 相同）
-Xmn1g                    # 新生代大小
-XX:+UseG1GC              # 使用 G1 收集器（JDK 9+ 默认）
-XX:MaxGCPauseMillis=200  # G1 目标停顿时间
-XX:+PrintGCDetails        # 打印 GC 日志
-XX:+HeapDumpOnOutOfMemoryError  # OOM 时 Dump
```

---

## 第5章 常用设计模式 ⭐

### 5.1 单例模式（大数据中最常见）

```java
// 双重检查锁（DCL）🔥
public class SparkSessionSingleton {
    private static volatile SparkSession instance;

    public static SparkSession getInstance(SparkConf conf) {
        if (instance == null) {
            synchronized (SparkSessionSingleton.class) {
                if (instance == null) {
                    instance = SparkSession.builder().config(conf).getOrCreate();
                }
            }
        }
        return instance;
    }
}
```

### 5.2 工厂模式

```java
// 简单工厂：根据类型创建不同的数据源读取器
public class ReaderFactory {
    public static DataReader createReader(String type) {
        switch (type) {
            case "mysql": return new MySQLReader();
            case "hdfs":  return new HDFSReader();
            case "kafka": return new KafkaReader();
            default: throw new IllegalArgumentException("Unknown type: " + type);
        }
    }
}
```

### 5.3 观察者模式

```java
// Flink 中的 Listener 机制就是观察者模式的应用
// 类似于 Kafka 消费者监听 Topic
```

---

## 第6章 面试要点 🔥🔥

### Q1：HashMap 和 ConcurrentHashMap 的区别？
> HashMap 线程不安全。ConcurrentHashMap：JDK7 分段锁，JDK8 CAS + synchronized（锁粒度为节点）。

### Q2：ArrayList 和 LinkedList 的区别？
> ArrayList 数组实现，随机访问快 O(1)，增删慢 O(n)。LinkedList 链表实现，增删快 O(1)，随机访问慢 O(n)。

### Q3：synchronized 和 ReentrantLock 的区别？
> synchronized 是关键字，自动释放。ReentrantLock 是类，需手动释放，支持公平锁、可中断、多条件变量。

### Q4：JVM 内存区域有哪些？
> **线程共享**：堆（对象实例）、方法区（类信息/常量池）。
> **线程私有**：虚拟机栈（局部变量/方法调用）、程序计数器、本地方法栈。

### Q5：什么时候触发 Full GC？
> 1. 老年代空间不足
> 2. 方法区空间不足
> 3. 调用 `System.gc()`（建议，非强制）
> 4. Minor GC 后存活对象大于老年代剩余空间

### Q6：Java 中的四种引用类型？
> | 引用 | 特点 | GC 行为 |
> |:---|:---|:---|
> | **强引用** | `Object obj = new Object()` | 不回收 |
> | **软引用** | `SoftReference` | 内存不足时回收 |
> | **弱引用** | `WeakReference` | 下次 GC 时回收 |
> | **虚引用** | `PhantomReference` | 随时回收 |

### Q7：大数据开发中常见的 OOM 场景？
> 1. Spark/Flink 处理大数据集，Executor 堆内存不足
> 2. 数据倾斜导致单 Task 处理过多数据
> 3. 大对象（大 Map/List）常驻内存
> 4. 解决：调大内存、优化数据倾斜、使用序列化框架（Kryo）
