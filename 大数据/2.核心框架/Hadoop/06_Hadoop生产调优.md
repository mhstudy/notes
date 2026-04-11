# Hadoop 生产调优

> 🔗 **官方文档**：https://hadoop.apache.org/docs/stable/
> 📌 **学习版本**：Hadoop 3.3.x

---

## 第1章 HDFS 核心参数调优 🔥

### 1.1 NameNode 内存配置 🔥

```bash
# hadoop-env.sh
# NN 内存 = 每百万个Block约占 1GB 内存
export HDFS_NAMENODE_OPTS="-Xmx4g"   # 小集群 4G
export HDFS_NAMENODE_OPTS="-Xmx64g"  # 大集群 64G+
```

> 💡 经验值：100万个文件块 ≈ 1GB 内存。生产中先 `hdfs dfsadmin -report` 查看 Block 数量。

### 1.2 NameNode 心跳并发配置

```xml
<!-- hdfs-site.xml -->
<!-- NN 处理 DN 心跳的线程数 -->
<property>
    <name>dfs.namenode.handler.count</name>
    <value>21</value>  <!-- 公式: 20 * ln(集群节点数)，如 20节点=60 -->
</property>
```

### 1.3 开启回收站 ⭐

```xml
<!-- core-site.xml -->
<property>
    <name>fs.trash.interval</name>
    <value>1440</value>  <!-- 文件保留 1440 分钟 = 1天 -->
</property>
<property>
    <name>fs.trash.checkpoint.interval</name>
    <value>1440</value>  <!-- 检查回收站间隔 -->
</property>
```

> ⚠️ 通过程序删除的文件不会进回收站，需要使用 `Trash` API 或在 Shell 中 `hadoop fs -rm` 才会进回收站。

---

## 第2章 HDFS 集群压测 📝

```bash
# 写性能测试
hadoop jar /opt/module/hadoop-3.3.4/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-3.3.4-tests.jar \
TestDFSIO -write -nrFiles 10 -fileSize 128MB

# 读性能测试
hadoop jar /opt/module/hadoop-3.3.4/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-3.3.4-tests.jar \
TestDFSIO -read -nrFiles 10 -fileSize 128MB

# 清理测试数据
hadoop jar ... TestDFSIO -clean
```

---

## 第3章 HDFS 集群扩容缩容 ⭐

### 3.1 白名单（允许连接 NN 的 DN）

```bash
# 创建白名单文件 $HADOOP_HOME/etc/hadoop/whitelist
hadoop102
hadoop103
hadoop104
```

```xml
<!-- hdfs-site.xml -->
<property>
    <name>dfs.hosts</name>
    <value>/opt/module/hadoop-3.3.4/etc/hadoop/whitelist</value>
</property>
```

### 3.2 黑名单退役（安全退役 DN）

```xml
<!-- hdfs-site.xml -->
<property>
    <name>dfs.hosts.exclude</name>
    <value>/opt/module/hadoop-3.3.4/etc/hadoop/blacklist</value>
</property>
```

```bash
# 刷新节点
hdfs dfsadmin -refreshNodes
yarn rmadmin -refreshNodes
```

### 3.3 数据均衡

```bash
# 节点间数据均衡
start-balancer.sh -threshold 10   # 偏差不超过10%
stop-balancer.sh

# 磁盘间数据均衡（单节点多磁盘）
hdfs diskbalancer -plan hadoop103
hdfs diskbalancer -execute hadoop103.plan.json
```

---

## 第4章 HDFS 存储优化 🔥

### 4.1 纠删码（Erasure Coding）🔥

Hadoop 3.x 新特性，**用计算换存储**，空间节省约 50%。

```
传统副本: 1份数据 × 3副本 = 3倍存储（存储利用率 33%）
纠删码:   RS-6-3-1024k → 6数据块 + 3校验块 = 1.5倍存储（利用率 67%）
```

```bash
# 查看当前策略
hdfs ec -listPolicies

# 设置纠删码策略
hdfs ec -setPolicy -path /input -policy RS-6-3-1024k

# 取消纠删码策略
hdfs ec -unsetPolicy -path /input
```

> ⚠️ 纠删码适合**冷数据**（不常访问），热数据还是用 3 副本更好（读取性能高）。

### 4.2 异构存储（冷热数据分离）⭐

| 策略 | 说明 | 适用场景 |
|:---|:---|:---|
| **HOT** | 全部存 DISK | 默认，常用数据 |
| **WARM** | 一份 DISK + 其余 ARCHIVE | 较少访问 |
| **COLD** | 全部 ARCHIVE | 极少访问（归档） |
| **ALL_SSD** | 全部 SSD | 高性能需求 |
| **LAZY_PERSIST** | 一份 RAM_DISK + 其余 DISK | 极高性能 |

```bash
# 设置存储策略
hdfs storagepolicies -setStoragePolicy -path /data/hot -policy HOT
hdfs storagepolicies -setStoragePolicy -path /data/cold -policy COLD
```

---

## 第5章 HDFS 故障排除 ⭐

### 5.1 安全模式

```bash
# 查看安全模式状态
hdfs dfsadmin -safemode get

# 手动进入/退出安全模式
hdfs dfsadmin -safemode enter
hdfs dfsadmin -safemode leave

# 等待安全模式退出
hdfs dfsadmin -safemode wait
```

> 安全模式下：只读不写。NN 启动时会自动进入安全模式，当 Block 上报率达到 99.9% 后自动退出。

### 5.2 小文件归档（HAR）🔥

```bash
# 将 /input 下的小文件归档
hadoop archive -archiveName input.har -p /input /output

# 查看归档文件
hadoop fs -ls har:///output/input.har

# 解压归档
hadoop fs -cp har:///output/input.har/* /output/unarchived
```

---

## 第6章 MapReduce 生产调优 🔥🔥

### 6.1 MR 跑的慢的原因

1. **计算机性能**：CPU、内存、磁盘、网络
2. **IO 操作**：数据倾斜、Map/Reduce 数不合理、小文件过多、溢写次数多、Merge 次数多
3. **代码问题**：未使用 Combiner、未使用 Map Join

### 6.2 常用调优参数 🔥

| 参数 | 默认值 | 说明 |
|:---|:---|:---|
| `mapreduce.map.memory.mb` | 1024 | Map Task 内存 |
| `mapreduce.reduce.memory.mb` | 1024 | Reduce Task 内存 |
| `mapreduce.map.cpu.vcores` | 1 | Map CPU 核数 |
| `mapreduce.reduce.cpu.vcores` | 1 | Reduce CPU 核数 |
| `mapreduce.task.io.sort.mb` | 100 | 环形缓冲区大小 |
| `mapreduce.map.sort.spill.percent` | 0.8 | 溢写阈值 |
| `mapreduce.task.io.sort.factor` | 10 | Merge 时同时打开文件数 |
| `mapreduce.map.output.compress` | false | Map 输出是否压缩 |

### 6.3 数据倾斜解决方案 🔥🔥

| 方案 | 适用场景 |
|:---|:---|
| 自定义 Partitioner | Key 分布不均 |
| 使用 Combiner | 预聚合减少数据量 |
| Map Join | 大小表 Join |
| 加随机前缀二次聚合 | 聚合计算倾斜 |
| 增加 ReduceTask 数 | 通用方案 |
| 采样打散 | Key 极度集中 |

---

## 第7章 YARN 生产调优 🔥

### 7.1 YARN 内存参数 🔥

```xml
<!-- yarn-site.xml -->
<!-- NM 可分配的总内存（物理内存的 80%） -->
<property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>65536</value>  <!-- 64GB -->
</property>

<!-- NM 可分配的 CPU 核数 -->
<property>
    <name>yarn.nodemanager.resource.cpu-vcores</name>
    <value>16</value>
</property>

<!-- 单个 Container 最大/最小内存 -->
<property>
    <name>yarn.scheduler.minimum-allocation-mb</name>
    <value>1024</value>  <!-- 最小 1GB -->
</property>
<property>
    <name>yarn.scheduler.maximum-allocation-mb</name>
    <value>16384</value>  <!-- 最大 16GB -->
</property>
```

### 7.2 容量调度器多队列 🔥

```xml
<!-- capacity-scheduler.xml -->
<property>
    <name>yarn.scheduler.capacity.root.queues</name>
    <value>default,hive,spark</value>
</property>
<property>
    <name>yarn.scheduler.capacity.root.default.capacity</name>
    <value>40</value>
</property>
<property>
    <name>yarn.scheduler.capacity.root.hive.capacity</name>
    <value>40</value>
</property>
<property>
    <name>yarn.scheduler.capacity.root.spark.capacity</name>
    <value>20</value>
</property>
```

---

## 第8章 Hadoop 小文件优化 🔥🔥

### 8.1 小文件弊端

- 每个文件/目录/Block 在 NN 占约 **150 字节**内存
- **100万个小文件** → NN 需要约 **150MB** 内存
- 10亿个小文件 → **150GB** 内存（NN 扛不住！）

### 8.2 解决方案

| 方案 | 说明 |
|:---|:---|
| **HAR 归档** | 将小文件打包成 HAR 文件 |
| **CombineTextInputFormat** | MR 切片时合并小文件 |
| **JVM 重用** | 减少 JVM 启动开销 |
| **Sequence File** | 将小文件作为 KV 存入 SequenceFile |
| **Hive 合并** | `set hive.merge.mapfiles=true` |

---

## 🔥 面试高频题

### Q1：Hadoop 小文件过多有什么影响？如何解决？
> 影响：大量小文件会占用 NameNode 大量内存（每个文件约 150 字节元数据）。解决：源头合并（采集时合并）、HAR 归档、CombineTextInputFormat、SequenceFile、Hive merge 参数。

### Q2：生产环境如何配置 YARN 资源？
> NM 可分配内存设为物理内存的 80%（预留给 OS），Container 最大内存根据任务需求设置（一般 4~16GB），CPU 核数根据实际情况调整。多租户场景使用容量调度器划分队列。

### Q3：Hadoop 3.x 纠删码的优缺点？
> 优点：存储空间节省约 50%（利用率从 33% 提升到 67%）。缺点：恢复数据需要计算，增加 CPU 开销和网络带宽；适合冷数据，不适合热数据。
