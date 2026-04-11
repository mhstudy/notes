# MapReduce

> 🔗 **官方文档**：https://hadoop.apache.org/docs/stable/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html
> 📌 **学习版本**：Hadoop 3.3.x

---

## 第1章 MapReduce 概述 ⭐

### 1.1 MapReduce 定义

MapReduce 是一个**分布式运算程序**的编程框架，将业务逻辑和框架结合在一起，在大规模集群上并行处理大数据集。

### 1.2 MapReduce 优缺点

| 优点 | 缺点 |
|:---|:---|
| 易于编程（实现 Mapper/Reducer） | 不擅长实时计算 |
| 良好的扩展性 | 不擅长流式计算 |
| 高容错性（Task 失败自动重试） | 不擅长 DAG（有向无环图）计算 |
| 适合大数据量离线处理 | 中间结果落磁盘，IO 开销大 |

### 1.3 MapReduce 核心思想 🔥🔥

```
输入数据
  │
  ▼
┌─────────────────────────────────────────────┐
│                  Map 阶段                    │
│  InputFormat → Split → Mapper → 分区排序      │
│  （并行处理，一个 Split 对应一个 MapTask）      │
└─────────────┬───────────────────────────────┘
              │ (Shuffle: 分区、排序、Combiner、压缩)
              ▼
┌─────────────────────────────────────────────┐
│                Reduce 阶段                   │
│  拉取数据 → 合并排序 → Reducer → OutputFormat  │
│  （一个分区对应一个 ReduceTask）                │
└─────────────────────────────────────────────┘
              │
              ▼
           输出数据
```

### 1.4 MapReduce 编程规范 🔥

| 组件 | 说明 |
|:---|:---|
| **Mapper** | 输入 KV → 业务逻辑 → 输出 KV |
| **Reducer** | 输入 Mapper 输出 → 汇总 → 输出 KV |
| **Driver** | 配置 Job 参数，提交 Job |

### 1.5 常用序列化类型 ⭐

| Java 类型 | Hadoop Writable 类型 |
|:---|:---|
| Boolean | BooleanWritable |
| Byte | ByteWritable |
| Int | IntWritable |
| Long | **LongWritable** |
| Float | FloatWritable |
| Double | DoubleWritable |
| String | **Text** |
| Map | MapWritable |
| Array | ArrayWritable |

---

## 第2章 WordCount 案例 🔥

### 2.1 Mapper

```java
public class WordCountMapper extends Mapper<LongWritable, Text, Text, IntWritable> {

    private Text outK = new Text();
    private IntWritable outV = new IntWritable(1);

    @Override
    protected void map(LongWritable key, Text value, Context context)
            throws IOException, InterruptedException {
        // 1. 获取一行
        String line = value.toString();
        // 2. 切割
        String[] words = line.split(" ");
        // 3. 循环写出
        for (String word : words) {
            outK.set(word);
            context.write(outK, outV);
        }
    }
}
```

### 2.2 Reducer

```java
public class WordCountReducer extends Reducer<Text, IntWritable, Text, IntWritable> {

    private IntWritable outV = new IntWritable();

    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context)
            throws IOException, InterruptedException {
        int sum = 0;
        // 累加
        for (IntWritable value : values) {
            sum += value.get();
        }
        outV.set(sum);
        context.write(key, outV);
    }
}
```

### 2.3 Driver

```java
public class WordCountDriver {
    public static void main(String[] args) throws Exception {
        // 1. 创建 Job
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf);

        // 2. 设置 Jar 路径
        job.setJarByClass(WordCountDriver.class);

        // 3. 关联 Mapper 和 Reducer
        job.setMapperClass(WordCountMapper.class);
        job.setReducerClass(WordCountReducer.class);

        // 4. 设置 Map 输出 KV 类型
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(IntWritable.class);

        // 5. 设置最终输出 KV 类型
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);

        // 6. 设置输入输出路径
        FileInputFormat.setInputPaths(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));

        // 7. 提交 Job
        boolean result = job.waitForCompletion(true);
        System.exit(result ? 0 : 1);
    }
}
```

---

## 第3章 Hadoop 序列化 ⭐

### 3.1 自定义 Bean 序列化

```java
public class FlowBean implements Writable {
    private long upFlow;
    private long downFlow;
    private long sumFlow;

    // 空参构造（反序列化必须）
    public FlowBean() {}

    @Override
    public void write(DataOutput out) throws IOException {
        out.writeLong(upFlow);
        out.writeLong(downFlow);
        out.writeLong(sumFlow);
    }

    @Override
    public void readFields(DataInput in) throws IOException {
        // 反序列化顺序必须和序列化一致
        this.upFlow = in.readLong();
        this.downFlow = in.readLong();
        this.sumFlow = in.readLong();
    }

    // getter、setter、toString ...
}
```

---

## 第4章 MapReduce 框架原理 🔥🔥

### 4.1 InputFormat 数据切片 🔥

```
文件大小            切片规则（默认 FileInputFormat）
300MB         →    切片1: 0~128MB
                   切片2: 128~256MB
                   切片3: 256~300MB  (剩余44MB < 128*1.1, 单独一片)

关键公式: splitSize = Math.max(minSize, Math.min(maxSize, blockSize))
默认 splitSize = blockSize = 128MB
```

> 💡 每个 Split 启动一个 MapTask，切片是逻辑划分，不物理分割文件。

**CombineTextInputFormat**：解决大量小文件场景，将多个小文件合并为一个切片。

```java
// Driver 中设置
job.setInputFormatClass(CombineTextInputFormat.class);
CombineTextInputFormat.setMaxInputSplitSize(job, 4194304); // 4MB
```

### 4.2 Shuffle 机制 🔥🔥🔥

Shuffle 是 MapReduce 的**核心**，从 Map 输出到 Reduce 输入的整个过程。

```
         Map 端                          Reduce 端
┌───────────────────┐             ┌───────────────────┐
│  Mapper 输出       │             │  Copy 阶段        │
│       │           │             │  从各 Map 拉取数据  │
│       ▼           │             │       │           │
│  环形缓冲区(100MB) │   ───→      │       ▼           │
│  达到80%溢写到磁盘  │   网络传输   │  Merge 归并排序    │
│       │           │             │       │           │
│       ▼           │             │       ▼           │
│  分区、排序        │             │  Reducer 处理      │
│  (可选 Combiner)  │             │       │           │
│       │           │             │       ▼           │
│       ▼           │             │  OutputFormat      │
│  Merge 合并溢写文件│             └───────────────────┘
└───────────────────┘
```

#### Partition 分区 🔥

默认分区规则：`HashPartitioner`

```java
// 默认分区器
public class HashPartitioner<K, V> extends Partitioner<K, V> {
    public int getPartition(K key, V value, int numReduceTasks) {
        return (key.hashCode() & Integer.MAX_VALUE) % numReduceTasks;
    }
}
```

自定义分区器：

```java
public class ProvincePartitioner extends Partitioner<Text, FlowBean> {
    @Override
    public int getPartition(Text key, FlowBean value, int numPartitions) {
        String phone = key.toString().substring(0, 3);
        switch (phone) {
            case "136": return 0;
            case "137": return 1;
            case "138": return 2;
            case "139": return 3;
            default:    return 4;
        }
    }
}
// Driver: job.setPartitionerClass(ProvincePartitioner.class);
//         job.setNumReduceTasks(5);
```

> ⚠️ 分区数 > ReduceTask 数：报错  
> 分区数 < ReduceTask 数：产生空文件  
> ReduceTask = 1：不走分区，结果只有一个文件

#### WritableComparable 排序 ⭐

```java
// 自定义排序：实现 WritableComparable 接口
public class FlowBean implements WritableComparable<FlowBean> {
    @Override
    public int compareTo(FlowBean o) {
        // 按总流量倒序
        return Long.compare(o.sumFlow, this.sumFlow);
    }
}
```

#### Combiner 合并 ⭐

Combiner 是 Map 端的**局部 Reducer**，减少网络传输。

```java
// 方式一：直接使用 Reducer 作为 Combiner
job.setCombinerClass(WordCountReducer.class);

// 方式二：自定义 Combiner
public class WordCountCombiner extends Reducer<Text, IntWritable, Text, IntWritable> {
    // 逻辑和 Reducer 一样，但在 Map 端执行
}
```

> ⚠️ Combiner 的**前提**：不能影响最终业务逻辑（如求平均值不能用 Combiner）

---

## 第5章 OutputFormat ⭐

| OutputFormat | 说明 |
|:---|:---|
| TextOutputFormat | 默认，按行输出 KV |
| SequenceFileOutputFormat | 二进制序列文件 |
| 自定义 OutputFormat | 控制输出目的地 |

---

## 第6章 Join 应用 🔥

### 6.1 Reduce Join

- 在 Map 端打标签，Reduce 端合并
- **缺点**：所有数据走网络传输到 Reduce 端，容易**数据倾斜**

### 6.2 Map Join 🔥

- 适用于**一张大表 + 一张小表**
- 在 Map 端缓存小表（`DistributedCache`），避免 Reduce

```java
// Driver 中加载缓存文件
job.addCacheFile(new URI("file:///opt/data/pd.txt"));
job.setNumReduceTasks(0);  // 不需要 Reduce

// Mapper 中 setup 读取缓存
@Override
protected void setup(Context context) throws IOException {
    URI[] cacheFiles = context.getCacheFiles();
    // 读取小表到 HashMap
    BufferedReader reader = new BufferedReader(
        new InputStreamReader(
            new FileInputStream(new File(new URI(cacheFiles[0]).getPath()))
        ));
    // ...
}
```

---

## 第7章 数据清洗（ETL）📝

ETL（Extract-Transform-Load）：在 MapReduce 中常用 Mapper 做数据清洗，不需要 Reducer。

```java
job.setNumReduceTasks(0);  // 仅 Map，不需要 Reduce
```

---

## 第8章 MapReduce 数据压缩 ⭐

| 压缩格式 | Hadoop 自带 | 是否可切分 | 压缩比 | 速度 |
|:---|:---|:---|:---|:---|
| gzip | 是 | **否** | 高 | 中 |
| bzip2 | 是 | **是** | 最高 | 慢 |
| LZO | 否（需安装） | **是（需索引）** | 中 | 快 |
| **Snappy** 🔥 | 是 | **否** | 中 | **最快** |

```java
// Map 输出压缩
conf.setBoolean("mapreduce.map.output.compress", true);
conf.setClass("mapreduce.map.output.compress.codec",
    SnappyCodec.class, CompressionCodec.class);

// Reduce 输出压缩
FileOutputFormat.setCompressOutput(job, true);
FileOutputFormat.setOutputCompressorClass(job, GzipCodec.class);
```

> 💡 **生产建议**：Map 输出用 **Snappy**（快），Reduce 输出用 **gzip/Snappy**（平衡压缩比和速度）

---

## 🔥 面试高频题

### Q1：请描述 MapReduce 的 Shuffle 过程？
> Map 端：Mapper 输出写入环形缓冲区（默认100MB），到达80%阈值溢写到磁盘（溢写前分区、排序），多次溢写的文件 Merge 合并。Reduce 端：Copy 线程从各 Map 拉取对应分区数据，Merge 归并排序，最终送入 Reducer 处理。

### Q2：MapReduce 数据倾斜如何解决？
> 1）自定义 Partitioner 均匀分区；2）使用 Combiner 预聚合；3）Map Join 代替 Reduce Join；4）增加 ReduceTask 数量；5）对倾斜 Key 加随机前缀打散再二次聚合。

### Q3：Map Join 和 Reduce Join 的区别？
> Map Join：小表缓存到内存，Map 端完成 Join，无 Reduce，适合大小表 Join。Reduce Join：所有数据到 Reduce 端合并，适合两张大表，但容易数据倾斜。

### Q4：MapReduce 的切片机制？
> 默认切片大小 = Block 大小 = 128MB。切片是逻辑划分，一个切片对应一个 MapTask。小文件场景可用 CombineTextInputFormat 合并切片。
