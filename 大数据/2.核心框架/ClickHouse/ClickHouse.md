# ClickHouse

> 🔗 **官方网站**：https://clickhouse.com/
> 📖 **官方文档**：https://clickhouse.com/docs
> 📌 **学习版本**：ClickHouse 23.x

---

## 第1章 ClickHouse 概述 ⭐

### 1.1 什么是 ClickHouse

ClickHouse 是由 Yandex 开发的**列式存储分析型数据库（OLAP）**，以极致的查询速度著称。

### 1.2 为什么快 🔥

| 技术 | 说明 |
|:---|:---|
| **列式存储** | 只读取查询需要的列，减少 IO |
| **数据压缩** | 同列数据类型相同，压缩比极高 |
| **向量化引擎** | 利用 SIMD 指令批量处理数据 |
| **多线程并行** | 充分利用多核 CPU |
| **稀疏索引** | 主键索引占用内存极少 |
| **数据分区** | 按分区裁剪，减少扫描量 |

---

## 第2章 数据类型与建表 ⭐

### 2.1 常用数据类型

| 类型 | 说明 |
|:---|:---|
| `UInt8/16/32/64` | 无符号整数 |
| `Int8/16/32/64` | 有符号整数 |
| `Float32/Float64` | 浮点数 |
| `Decimal(P,S)` | 高精度数字 |
| `String` | 变长字符串 |
| `FixedString(N)` | 定长字符串 |
| `Date / DateTime` | 日期 / 日期时间 |
| `Array(T)` | 数组 |
| `Nullable(T)` | 可空类型 |
| `Enum8/Enum16` | 枚举类型 |

### 2.2 表引擎 🔥🔥

#### MergeTree 引擎族（核心！）

| 引擎 | 说明 | 适用场景 |
|:---|:---|:---|
| **MergeTree** 🔥 | 基础引擎，支持主键排序、分区 | 通用场景 |
| **ReplacingMergeTree** | 去重（按主键，合并时去重） | 需要去重 |
| **SummingMergeTree** | 预聚合（按主键，合并时求和） | 指标汇总 |
| **AggregatingMergeTree** | 增量聚合 | 物化视图 |
| **CollapsingMergeTree** | 折叠树（用 sign 列标记删改） | 需要更新 |
| **VersionedCollapsingMergeTree** | 版本折叠树 | 乱序更新 |

```sql
-- MergeTree 建表示例
CREATE TABLE visits (
    id UInt64,
    user_id UInt32,
    url String,
    duration UInt32,
    sign Int8,
    event_date Date,
    event_time DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)    -- 按月分区
ORDER BY (user_id, event_time)        -- 排序键（也是稀疏索引）
TTL event_date + INTERVAL 3 MONTH    -- 数据3个月过期
SETTINGS index_granularity = 8192;    -- 索引粒度
```

```sql
-- ReplacingMergeTree（去重）
CREATE TABLE user_info (
    user_id UInt32,
    name String,
    age UInt8,
    update_time DateTime
)
ENGINE = ReplacingMergeTree(update_time)  -- 按 update_time 保留最新
ORDER BY user_id;

-- SummingMergeTree（预聚合）
CREATE TABLE daily_stats (
    event_date Date,
    page String,
    pv UInt64,
    uv UInt64
)
ENGINE = SummingMergeTree((pv, uv))  -- pv, uv 自动求和
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, page);
```

---

## 第3章 SQL 查询 🔥

```sql
-- 基本查询
SELECT user_id, COUNT(*) AS cnt, SUM(duration) AS total_dur
FROM visits
WHERE event_date >= '2023-01-01'
GROUP BY user_id
ORDER BY cnt DESC
LIMIT 10;

-- 数组操作
SELECT arrayJoin([1, 2, 3]) AS num;
SELECT groupArray(name) FROM users GROUP BY dept;

-- 窗口函数
SELECT user_id, event_time, duration,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY event_time) AS rn,
    SUM(duration) OVER (PARTITION BY user_id ORDER BY event_time) AS cum_dur
FROM visits;

-- 物化视图（自动聚合）🔥
CREATE MATERIALIZED VIEW daily_pv_mv
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, url)
AS SELECT
    toDate(event_time) AS event_date,
    url,
    COUNT() AS pv
FROM visits
GROUP BY event_date, url;
```

---

## 第4章 集群架构 ⭐

### 4.1 副本与分片

```
┌─────────────────────────────────────────────┐
│              ClickHouse Cluster              │
├────────────────┬────────────────────────────┤
│   Shard 1      │    Shard 2                 │
│ ┌────┐ ┌────┐ │ ┌────┐ ┌────┐              │
│ │ R1 │ │ R2 │ │ │ R1 │ │ R2 │              │
│ └────┘ └────┘ │ └────┘ └────┘              │
│  (主)   (副本) │  (主)   (副本)              │
└────────────────┴────────────────────────────┘
```

| 概念 | 说明 |
|:---|:---|
| **Shard（分片）** | 数据水平切分，不同分片存不同数据 |
| **Replica（副本）** | 同一分片的数据副本，高可用 |
| **ReplicatedMergeTree** | 副本引擎，通过 ZooKeeper 同步 |
| **Distributed** | 分布式表引擎，查询路由到各分片 |

### 4.2 分布式表

```sql
-- 本地表（每个节点上）
CREATE TABLE visits_local ON CLUSTER my_cluster (
    id UInt64,
    user_id UInt32,
    url String,
    event_date Date
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/visits', '{replica}')
PARTITION BY toYYYYMM(event_date)
ORDER BY (user_id, event_date);

-- 分布式表（路由层）
CREATE TABLE visits_all ON CLUSTER my_cluster AS visits_local
ENGINE = Distributed(my_cluster, default, visits_local, rand());
```

---

## 第5章 数据导入与导出 ⭐

```bash
# 从 CSV 导入
clickhouse-client --query="INSERT INTO visits FORMAT CSV" < data.csv

# 从 HDFS 导入
INSERT INTO visits
SELECT * FROM hdfs('hdfs://hadoop102:8020/data/visits/*.parquet', 'Parquet');

# 从 Kafka 实时消费 🔥
CREATE TABLE visits_kafka (
    id UInt64,
    user_id UInt32,
    url String
) ENGINE = Kafka()
SETTINGS
    kafka_broker_list = 'hadoop102:9092',
    kafka_topic_list = 'visits_topic',
    kafka_group_name = 'ch_consumer',
    kafka_format = 'JSONEachRow';

-- 物化视图自动消费写入
CREATE MATERIALIZED VIEW visits_consumer TO visits AS
SELECT * FROM visits_kafka;
```

---

## 第6章 性能优化 🔥🔥

### 6.1 建表优化

| 优化项 | 建议 |
|:---|:---|
| **ORDER BY** | 查询频繁的过滤列放前面 |
| **PARTITION BY** | 按时间分区，通常按月 `toYYYYMM()` |
| **数据类型** | 用最小的类型（UInt8 优于 UInt64） |
| **TTL** | 设置过期自动删除 |
| **索引粒度** | 默认 8192，小表可适当减小 |

### 6.2 查询优化

```sql
-- ✅ 使用 PREWHERE 替代 WHERE（自动优化，也可手动指定）
SELECT * FROM visits PREWHERE user_id = 1001 WHERE duration > 100;

-- ✅ 避免 SELECT *，只查需要的列
SELECT user_id, url FROM visits WHERE event_date = '2023-06-15';

-- ✅ 使用近似计算函数
SELECT uniqCombined(user_id) AS approx_uv FROM visits;  -- 比 COUNT(DISTINCT) 快

-- ✅ 使用 IN 替代 JOIN（小维表场景）
SELECT * FROM visits WHERE user_id IN (SELECT user_id FROM vip_users);
```

### 6.3 常用配置

```xml
<!-- config.xml 关键配置 -->
<max_memory_usage>10000000000</max_memory_usage>          <!-- 单查询内存 10GB -->
<max_threads>8</max_threads>                               <!-- 查询线程数 -->
<max_insert_block_size>1048576</max_insert_block_size>    <!-- 写入块大小 -->
<merge_tree>
    <max_bytes_to_merge_at_max_space_in_pool>161061273600</max_bytes_to_merge_at_max_space_in_pool>
</merge_tree>
```

---

## 第7章 高频面试题 🔥🔥🔥

### Q1：ClickHouse 为什么这么快？
> 列式存储 + 数据压缩 + 向量化引擎 + 多线程并行 + 稀疏索引 + 分区裁剪

### Q2：ClickHouse 和 Doris 的区别？
> | 对比 | ClickHouse | Doris |
> |:---|:---|:---|
> | 架构 | 单机性能极强，集群需手动分片 | **MPP 架构**，自动分布式 |
> | Join | 大表 Join 较弱 | **支持大表 Join** |
> | 更新 | 有限支持 | 支持 Unique Key 更新 |
> | 运维 | 复杂 | **简单** |
> | 适合 | 单表大查询、宽表 | 多表关联分析 |

### Q3：MergeTree 引擎的特点？
> 支持分区、主键排序、稀疏索引、数据 TTL、后台自动 Merge。是 ClickHouse 最核心的引擎族。

### Q4：ReplacingMergeTree 什么时候去重？
> **合并时去重**，不是实时去重！查询时可能看到重复数据，需要用 `FINAL` 关键字或 `argMax` 函数保证去重。

### Q5：ClickHouse 如何处理数据更新？
> ClickHouse 不擅长频繁更新。解决方案：
> 1. **ReplacingMergeTree** + `FINAL` 查询
> 2. **CollapsingMergeTree** 通过 sign 列标记删改
> 3. **Mutation**（ALTER TABLE UPDATE/DELETE），异步执行，不适合高频操作

### Q6：ClickHouse 的稀疏索引原理？
> 每隔 `index_granularity`（默认8192）行记录一个索引标记，索引文件极小可常驻内存。查询时通过二分查找快速定位数据块范围，大幅减少 IO。

### Q7：如何优化 ClickHouse 查询性能？
> 1. 合理设计 ORDER BY（高频过滤列放前）
> 2. 避免 `SELECT *`，只查需要列
> 3. 使用物化视图预聚合
> 4. 用 `uniqCombined` 替代 `COUNT(DISTINCT)`
> 5. 小表用 IN 替代 JOIN
> 6. 使用 PREWHERE 提前过滤
