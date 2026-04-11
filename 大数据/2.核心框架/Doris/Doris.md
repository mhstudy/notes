# Doris

> 🔗 **官方网站**：https://doris.apache.org/
> 📖 **官方文档**：https://doris.apache.org/docs/
> 📌 **学习版本**：Apache Doris 2.x

---

## 第1章 Doris 概述 ⭐

### 1.1 什么是 Doris

Apache Doris 是一个**MPP 分析型数据库**，支持实时数据分析，兼容 MySQL 协议，以极简架构和高性能著称。

### 1.2 架构 🔥

```
┌──────────────────────────────────┐
│           FE (Frontend)           │
│  ● SQL 解析  ● 查询规划           │
│  ● 元数据管理  ● 调度              │
└──────────┬───────────────────────┘
           │
    ┌──────┼──────┐
    ▼      ▼      ▼
┌──────┐┌──────┐┌──────┐
│  BE  ││  BE  ││  BE  │
│(数据) ││(数据) ││(数据) │
└──────┘└──────┘└──────┘
```

| 组件 | 说明 |
|:---|:---|
| **FE（Frontend）** | 接收请求、解析 SQL、查询规划、元数据管理 |
| **BE（Backend）** | 存储数据、执行计算 |

**特点**：只有 FE + BE 两个进程，架构极简，运维方便。

---

## 第2章 数据模型 🔥

| 模型 | 说明 | 适用场景 |
|:---|:---|:---|
| **Aggregate（聚合模型）** | 相同 Key 的行自动聚合（SUM/MAX/MIN/REPLACE）| 指标汇总 |
| **Unique（唯一模型）** 🔥 | 相同 Key 保留最新值（REPLACE） | 用户画像、维度表 |
| **Duplicate（明细模型）** | 不做任何聚合，保留所有数据 | 日志明细 |

```sql
-- Unique 模型（最常用）
CREATE TABLE user_profile (
    user_id BIGINT,
    name VARCHAR(50),
    age INT,
    city VARCHAR(50),
    update_time DATETIME
)
UNIQUE KEY(user_id)
DISTRIBUTED BY HASH(user_id) BUCKETS 8
PROPERTIES (
    "replication_num" = "3",
    "enable_unique_key_merge_on_write" = "true"  -- 写时合并，查询更快
);

-- Aggregate 模型
CREATE TABLE site_pv (
    dt DATE,
    page VARCHAR(200),
    pv BIGINT SUM,       -- 自动求和
    uv BIGINT BITMAP_UNION  -- Bitmap 去重
)
AGGREGATE KEY(dt, page)
DISTRIBUTED BY HASH(dt) BUCKETS 4;

-- Duplicate 模型
CREATE TABLE event_log (
    event_time DATETIME,
    user_id BIGINT,
    event_type VARCHAR(50),
    event_data VARCHAR(1000)
)
DUPLICATE KEY(event_time, user_id)
DISTRIBUTED BY HASH(user_id) BUCKETS 8;
```

---

## 第3章 数据导入 ⭐

```sql
-- Stream Load（推荐，实时导入）
curl --location-trusted -u root: \
    -H "format: json" \
    -H "strip_outer_array: true" \
    -T data.json \
    http://hadoop102:8030/api/gmall/user_profile/_stream_load

-- Broker Load（大批量从 HDFS 导入）
LOAD LABEL gmall.load_20230615
(
    DATA INFILE("hdfs://hadoop102:8020/data/user_info/*")
    INTO TABLE user_profile
    FORMAT AS "orc"
)
WITH BROKER "hdfs_broker";

-- INSERT INTO
INSERT INTO user_profile SELECT * FROM source_table;

-- Routine Load（从 Kafka 持续导入）
CREATE ROUTINE LOAD gmall.user_load ON user_profile
PROPERTIES ("format" = "json", "desired_concurrent_number" = "3")
FROM KAFKA (
    "kafka_broker_list" = "hadoop102:9092",
    "kafka_topic" = "user_topic"
);
```

### 导入方式对比 🔥

| 导入方式 | 适用场景 | 实时性 | 数据源 |
|:---|:---|:---|:---|
| **Stream Load** 🔥 | 小批量、微批实时 | 秒级 | 本地文件/程序推送 |
| **Broker Load** | 大批量离线导入 | 分钟级 | HDFS/S3 |
| **Routine Load** | 持续流式导入 | 秒级 | Kafka |
| **INSERT INTO** | SQL 导入 | — | 内部表查询 |

---

## 第4章 分区与分桶 🔥🔥

### 4.1 分区（Partition）

```sql
-- Range 分区（按时间）
CREATE TABLE order_info (
    order_id BIGINT,
    user_id BIGINT,
    amount DECIMAL(10,2),
    order_date DATE
)
UNIQUE KEY(order_id)
PARTITION BY RANGE(order_date) (
    PARTITION p202301 VALUES LESS THAN ("2023-02-01"),
    PARTITION p202302 VALUES LESS THAN ("2023-03-01"),
    PARTITION p202303 VALUES LESS THAN ("2023-04-01")
)
DISTRIBUTED BY HASH(order_id) BUCKETS 8;

-- 动态分区（自动管理）🔥
CREATE TABLE event_log (...)
PARTITION BY RANGE(event_date) ()
PROPERTIES (
    "dynamic_partition.enable" = "true",
    "dynamic_partition.time_unit" = "DAY",
    "dynamic_partition.start" = "-30",     -- 保留30天前
    "dynamic_partition.end" = "3",         -- 预创建3天
    "dynamic_partition.prefix" = "p",
    "dynamic_partition.buckets" = "8"
);
```

### 4.2 分桶（Bucket）

分桶是 Doris 数据分布的最小单元：

| 分桶策略 | 说明 |
|:---|:---|
| `HASH(col)` | 按指定列 Hash 分桶，常用 |
| `RANDOM` | 随机分桶（Doris 2.x 新增） |

**分桶数建议**：`单分区数据量 / (1~2 GB)` ≈ 桶数

---

## 第5章 物化视图与 Rollup ⭐

### 5.1 Rollup（上卷表）

```sql
-- 在 order_info 上创建 Rollup
ALTER TABLE order_info ADD ROLLUP rollup_user_amount(user_id, amount);

-- 查询时自动命中 Rollup（无需修改 SQL）
SELECT user_id, SUM(amount) FROM order_info GROUP BY user_id;
```

### 5.2 物化视图

```sql
-- 创建同步物化视图
CREATE MATERIALIZED VIEW mv_daily_amount AS
SELECT order_date, user_id, SUM(amount) AS total_amount, COUNT(*) AS order_cnt
FROM order_info
GROUP BY order_date, user_id;

-- 查询自动路由到物化视图
SELECT order_date, SUM(total_amount) FROM order_info GROUP BY order_date;
```

---

## 第6章 查询优化 🔥

### 6.1 Join 优化

```sql
-- Doris 支持的 Join 类型
-- 1. Broadcast Join（小表广播，默认小表 < 1024MB）
-- 2. Shuffle Join（大表 Hash 重分布）

-- 强制使用 Broadcast Join
SELECT /*+ SET_VAR(exec_mem_limit=8589934592) */ *
FROM big_table a
JOIN [broadcast] small_table b ON a.id = b.id;

-- Colocation Join（同一组表数据本地化，避免 Shuffle）🔥
CREATE TABLE table_a (...) 
PROPERTIES ("colocate_with" = "group1");

CREATE TABLE table_b (...)
PROPERTIES ("colocate_with" = "group1");
```

### 6.2 查询配置优化

```sql
-- 常用优化参数
SET exec_mem_limit = 8589934592;           -- 单查询内存 8GB
SET parallel_fragment_exec_instance_num = 8; -- 并行度
SET enable_profile = true;                  -- 开启 Profile 分析
SET batch_size = 4096;                      -- 向量化批大小
```

---

## 第7章 面试题 🔥🔥

### Q1：Doris 和 ClickHouse 的区别？
> Doris：MPP 架构，支持大表 Join，运维简单，兼容 MySQL。
> ClickHouse：单机性能极强，大表 Join 较弱，运维复杂。
> **简单说**：多表关联选 Doris，单表大查询选 ClickHouse。

### Q2：Doris 的三种数据模型？
> Aggregate（聚合）、Unique（唯一，最常用）、Duplicate（明细）。各有适用场景，Unique 模型写时合并（merge-on-write）模式查询性能最好。

### Q3：Doris 为什么查询快？
> 列式存储 + 向量化引擎 + MPP 并行 + 智能物化视图 + CBO 优化器 + Colocation Join。

### Q4：Doris 的分区和分桶有什么区别？
> **分区**：数据的第一级划分，通常按时间 Range 分区，支持动态管理。
> **分桶**：分区内的数据分布方式，按指定列 Hash 分桶，是数据的最小管理单元。
> 查询先做分区裁剪，再做分桶裁剪，两级过滤减少扫描量。

### Q5：Routine Load 导入数据丢失或重复怎么办？
> 丢失：检查 Kafka offset 提交是否正常，确认 Routine Load 任务状态。
> 重复：Unique 模型天然去重；Duplicate 模型需要业务层去重。
> 建议使用 Unique 模型 + merge-on-write 保证幂等性。

### Q6：Doris 如何处理大表 Join？
> 1. 小表使用 Broadcast Join（自动判断或手动指定）
> 2. 大表使用 Shuffle Join + Colocation Join 优化
> 3. 使用 Rollup/物化视图预聚合减少 Join 数据量
