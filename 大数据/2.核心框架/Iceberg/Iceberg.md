# Iceberg

> Apache Iceberg 官网：https://iceberg.apache.org/
> 官方文档：https://iceberg.apache.org/docs/latest/
>
> 本文基于 Iceberg 1.1.0 版本

---

## 第1章 Iceberg 简介

### 1.1 🔥 概述

#### 为什么需要数据湖

传统数据仓库（Hive）的痛点：
- **Schema 变更困难**：加列、改类型需要重写整张表
- **分区变更困难**：修改分区策略需要重写数据 + 迁移
- **无法高效更新/删除**：Hive 的 ACID 性能差
- **无法保证一致性读**：写入过程中可能读到脏数据
- **无法进行时间旅行**：无法查询历史版本数据

#### Iceberg 是什么

Apache Iceberg 是一种**开放的表格式**（Table Format），为大型数据集提供高性能的读写和元数据管理。

> 🔥 **Iceberg 不是存储引擎，不是计算引擎**，而是介于计算引擎（Spark/Flink/Trino）和存储格式（Parquet/ORC/Avro）之间的一个**中间层**。

### 1.2 🔥 核心特性

| 特性 | 说明 | 优先级 |
|------|------|--------|
| **计算引擎插件化** | 支持 Spark、Flink、Trino、Hive 等多种引擎 | 🔥 |
| **流批一体** | 支持批量读写和流式读写 | 🔥 |
| **Schema Evolution** | 🔥 支持加列、删列、重命名列、改类型，**无需重写数据** | 🔥 |
| **Partition Evolution** | 🔥 支持修改分区策略，**历史数据不受影响** | 🔥 |
| **Hidden Partition** | 分区对用户透明，查询不需要指定分区列 | ⭐ |
| **Time Travel** | 🔥 支持查询历史快照数据 | 🔥 |
| **ACID 事务** | 支持并发读写的事务隔离 | 🔥 |
| **乐观并发控制** | 基于乐观锁的并发支持 | ⭐ |
| **文件级数据剪裁** | 利用列级统计信息跳过无关文件 | ⭐ |

### 1.3 ⭐ 数据湖框架对比

| 对比项 | **Iceberg** | **Delta Lake** | **Hudi** |
|--------|------------|---------------|---------|
| 开源社区 | Apache 顶级项目 | Databricks 主导 | Uber → Apache |
| 计算引擎 | 🔥 Spark/Flink/Trino/Hive | Spark（其他支持弱） | Spark/Flink |
| Schema Evolution | ✅ 完善 | ✅ 基本支持 | ⚠️ 有限 |
| Partition Evolution | ✅ 支持 | ❌ 不支持 | ❌ 不支持 |
| Time Travel | ✅ 基于快照 | ✅ 基于版本 | ✅ 基于时间线 |
| 流式写入 | ✅ | ⚠️ Spark 为主 | ✅ 原生支持 |
| 更新/删除 | COW + MOR | COW + DV | COW + MOR |
| 社区活跃度 | 🔥 非常活跃 | 活跃 | 活跃 |

---

## 第2章 存储结构

### 2.1 🔥 整体架构

Iceberg 的元数据采用**树形结构**，从上到下分为三层：

```
Catalog
  └── Table Metadata（元数据文件）
        ├── Snapshot 1
        │     └── Manifest List 1
        │           ├── Manifest File 1 → [data file 1, data file 2]
        │           └── Manifest File 2 → [data file 3]
        └── Snapshot 2（当前快照）
              └── Manifest List 2
                    ├── Manifest File 1 → [data file 1, data file 2]（复用）
                    ├── Manifest File 2 → [data file 4]（新增）
                    └── Manifest File 3 → [data file 5]（新增）
```

### 2.2 📝 各层详解

| 层级 | 文件类型 | 存储内容 | 格式 |
|------|---------|---------|------|
| **数据层** | Data Files | 实际数据 | Parquet / ORC / Avro |
| **清单文件** | Manifest File | 🔥 数据文件列表 + 每个文件的列级统计信息（min/max/count/null_count） | Avro |
| **清单列表** | Manifest List | 🔥 所有 Manifest File 的列表 + 分区统计信息 | Avro |
| **快照** | Snapshot | 指向 Manifest List 的指针 + 快照时间 + 操作类型 | JSON（metadata file） |

> 🔥 **核心优势**：通过 Manifest File 中的**列级统计信息**（min/max），查询时可以**跳过不需要读取的文件**，大幅提升查询性能。

### 2.3 🔥 快照（Snapshot）机制

- 每次写入操作（INSERT/UPDATE/DELETE）都会生成一个**新的快照**
- 快照是**不可变的**（Immutable），支持 Time Travel
- 新快照会**复用**未被修改的 Manifest File，减少元数据开销
- 通过 `snapshot-id` 可以查询任意历史版本的数据

---

## 第3章 与 Hive 集成

### 3.1 ⭐ Catalog 配置

```sql
-- 默认使用 HiveCatalog
SET iceberg.catalog.default.type=hive;
SET iceberg.catalog.default.uri=thrift://hadoop102:9083;
SET iceberg.catalog.default.warehouse=hdfs://hadoop102:8020/user/hive/warehouse;

-- 创建 Iceberg 表
CREATE TABLE iceberg_db.sample (
    id BIGINT,
    name STRING,
    ts TIMESTAMP
) STORED BY 'org.apache.iceberg.mr.hive.HiveIcebergStorageHandler';
```

### 3.2 📝 基本操作

```sql
-- 插入数据
INSERT INTO iceberg_db.sample VALUES (1, 'Alice', '2024-01-01 00:00:00');

-- 查询数据
SELECT * FROM iceberg_db.sample;

-- Schema Evolution（加列）
ALTER TABLE iceberg_db.sample ADD COLUMNS (age INT);

-- Time Travel（按快照查询）
SELECT * FROM iceberg_db.sample FOR SYSTEM_VERSION AS OF 1234567890;
```

---

## 第4章 与 Spark SQL 集成

### 4.1 🔥 环境配置

```bash
# 启动 spark-sql 并加载 Iceberg
spark-sql --packages org.apache.iceberg:iceberg-spark-runtime-3.3_2.12:1.1.0 \
    --conf spark.sql.catalog.hadoop_catalog=org.apache.iceberg.spark.SparkCatalog \
    --conf spark.sql.catalog.hadoop_catalog.type=hadoop \
    --conf spark.sql.catalog.hadoop_catalog.warehouse=hdfs://hadoop102:8020/iceberg/warehouse \
    --conf spark.sql.catalog.hive_catalog=org.apache.iceberg.spark.SparkCatalog \
    --conf spark.sql.catalog.hive_catalog.type=hive \
    --conf spark.sql.catalog.hive_catalog.uri=thrift://hadoop102:9083
```

### 4.2 🔥 SQL 操作

```sql
-- 创建表
CREATE TABLE hadoop_catalog.db.sample (
    id BIGINT,
    name STRING,
    ts TIMESTAMP
) USING iceberg
PARTITIONED BY (days(ts));  -- 隐藏分区：按天分区

-- 插入数据
INSERT INTO hadoop_catalog.db.sample VALUES
    (1, 'Alice', TIMESTAMP '2024-01-01 10:00:00'),
    (2, 'Bob', TIMESTAMP '2024-01-02 10:00:00');

-- 🔥 Time Travel（按快照ID查询）
SELECT * FROM hadoop_catalog.db.sample VERSION AS OF 123456789;

-- 🔥 Time Travel（按时间查询）
SELECT * FROM hadoop_catalog.db.sample TIMESTAMP AS OF '2024-01-01 00:00:00';

-- 🔥 Schema Evolution
ALTER TABLE hadoop_catalog.db.sample ADD COLUMN age INT;
ALTER TABLE hadoop_catalog.db.sample RENAME COLUMN name TO user_name;
ALTER TABLE hadoop_catalog.db.sample ALTER COLUMN id TYPE BIGINT;

-- 🔥 Partition Evolution（无需重写数据！）
ALTER TABLE hadoop_catalog.db.sample ADD PARTITION FIELD bucket(16, id);
ALTER TABLE hadoop_catalog.db.sample DROP PARTITION FIELD days(ts);

-- 查看快照历史
SELECT * FROM hadoop_catalog.db.sample.snapshots;

-- 查看数据文件
SELECT * FROM hadoop_catalog.db.sample.files;

-- 查看操作历史
SELECT * FROM hadoop_catalog.db.sample.history;
```

### 4.3 🔥 存储过程

```sql
-- 过期快照清理（保留最近3天的快照）
CALL hadoop_catalog.system.expire_snapshots('db.sample', TIMESTAMP '2024-01-01 00:00:00', 100);

-- 删除孤立文件
CALL hadoop_catalog.system.remove_orphan_files('db.sample');

-- 合并小文件（compaction）
CALL hadoop_catalog.system.rewrite_data_files('db.sample');

-- 回滚到指定快照
CALL hadoop_catalog.system.rollback_to_snapshot('db.sample', 123456789);

-- 合并清单文件
CALL hadoop_catalog.system.rewrite_manifests('db.sample');
```

### 4.4 🔥 COW vs MOR

| 对比项 | **Copy-On-Write (COW)** | **Merge-On-Read (MOR)** |
|--------|------------------------|------------------------|
| **写入方式** | 重写整个数据文件 | 写入 Delete File + 新数据文件 |
| **读取方式** | 直接读取 | 🔥 读取时合并 Delete File |
| **写入性能** | 慢（需要重写） | 🔥 快 |
| **读取性能** | 🔥 快（无需合并） | 慢（需要合并） |
| **适用场景** | 读多写少 | 🔥 写多读少、实时场景 |
| **Format Version** | v1（默认） | v2 |

```sql
-- 创建 MOR 表（v2 格式）
CREATE TABLE hadoop_catalog.db.mor_table (
    id BIGINT,
    name STRING
) USING iceberg
TBLPROPERTIES (
    'format-version' = '2',
    'write.delete.mode' = 'merge-on-read',
    'write.update.mode' = 'merge-on-read',
    'write.merge.mode' = 'merge-on-read'
);
```

---

## 第5章 与 Flink SQL 集成

### 5.1 🔥 环境准备

```bash
# 将 Iceberg Flink runtime jar 放到 Flink lib 目录
cp iceberg-flink-runtime-1.17-1.1.0.jar $FLINK_HOME/lib/

# 启动 sql-client
bin/sql-client.sh
```

### 5.2 ⭐ Catalog 配置

```sql
-- 创建 Hive Catalog
CREATE CATALOG hive_catalog WITH (
    'type' = 'iceberg',
    'catalog-type' = 'hive',
    'uri' = 'thrift://hadoop102:9083',
    'warehouse' = 'hdfs://hadoop102:8020/iceberg/warehouse'
);

-- 创建 Hadoop Catalog
CREATE CATALOG hadoop_catalog WITH (
    'type' = 'iceberg',
    'catalog-type' = 'hadoop',
    'warehouse' = 'hdfs://hadoop102:8020/iceberg/warehouse'
);

USE CATALOG hive_catalog;
```

### 5.3 🔥 DDL 操作

```sql
-- 创建数据库
CREATE DATABASE iceberg_db;
USE iceberg_db;

-- 创建表
CREATE TABLE sample (
    id BIGINT,
    name STRING,
    ts TIMESTAMP(3),
    PRIMARY KEY (id) NOT ENFORCED
) WITH (
    'format-version' = '2',
    'write.upsert.enabled' = 'true'
);

-- 加列
ALTER TABLE sample ADD (age INT);

-- 修改列
ALTER TABLE sample MODIFY (name STRING COMMENT '用户名');
```

### 5.4 🔥 数据写入

```sql
-- INSERT INTO
INSERT INTO sample VALUES (1, 'Alice', TIMESTAMP '2024-01-01 10:00:00', 25);

-- INSERT OVERWRITE
INSERT OVERWRITE sample VALUES (1, 'Alice_new', TIMESTAMP '2024-01-01 10:00:00', 26);

-- 🔥 UPSERT（需要主键 + format-version=2 + write.upsert.enabled=true）
INSERT INTO sample VALUES (1, 'Alice_updated', TIMESTAMP '2024-01-01 10:00:00', 27);
```

### 5.5 🔥 流式读写

```sql
-- 🔥 Streaming 模式读取 Iceberg 表（增量读取）
SET 'execution.runtime-mode' = 'streaming';

SELECT * FROM sample /*+ OPTIONS('streaming'='true', 'monitor-interval'='5s') */;

-- 流式写入（从 Kafka 读取写入 Iceberg）
INSERT INTO sample
SELECT
    CAST(JSON_VALUE(data, '$.id') AS BIGINT),
    JSON_VALUE(data, '$.name'),
    TO_TIMESTAMP(JSON_VALUE(data, '$.ts')),
    CAST(JSON_VALUE(data, '$.age') AS INT)
FROM kafka_source;
```

### 5.6 📝 Flink 集成的不足

- Flink + Iceberg 的 **流式读取** 目前不支持 Watermark 推进
- **DELETE/UPDATE** SQL 在 Flink 中支持有限（需要 v2 格式 + 特定配置）
- 小文件合并需要单独的 Flink 任务或 Spark 存储过程

---

## 第6章 与 Flink DataStream 集成

### 6.1 ⭐ Maven 依赖

```xml
<dependency>
    <groupId>org.apache.iceberg</groupId>
    <artifactId>iceberg-flink-runtime-1.17</artifactId>
    <version>1.1.0</version>
</dependency>
```

### 6.2 📝 读写数据

```java
// 读取 Iceberg 表
TableLoader tableLoader = TableLoader.fromHadoopTable("hdfs://hadoop102:8020/iceberg/warehouse/db/sample");

DataStream<RowData> stream = FlinkSource.forRowData()
    .env(env)
    .tableLoader(tableLoader)
    .streaming(true)         // 流式读取
    .monitorInterval(Duration.ofSeconds(5))  // 监控间隔
    .build();

// 写入 Iceberg 表
FlinkSink.forRowData(inputStream)
    .tableLoader(tableLoader)
    .overwrite(false)
    .build();
```

### 6.3 ⭐ 合并小文件

```java
// 使用 Spark 存储过程合并小文件（推荐）
// CALL catalog.system.rewrite_data_files('db.sample')

// 也可以通过 Flink 任务定期执行 Compaction
Actions.forTable(table)
    .rewriteDataFiles()
    .targetSizeInBytes(128 * 1024 * 1024)  // 128MB
    .execute();
```

---

## 🔥 Iceberg 面试高频问题

### Q1：Iceberg 的存储结构是怎样的？

**三层元数据 + 数据文件**：
1. **Metadata File**：记录 Schema、分区策略、快照列表
2. **Manifest List**：记录所有 Manifest File 的清单 + 分区统计
3. **Manifest File**：记录数据文件列表 + 列级统计信息（min/max/null_count）
4. **Data Files**：实际数据（Parquet/ORC/Avro）

🔥 **查询优化**：利用 Manifest File 中的列级统计信息，实现**文件级数据剪裁**，跳过不需要读取的文件。

### Q2：Iceberg 如何实现 Schema Evolution？

- **不需要重写数据**！
- 通过在 Metadata 中记录 Schema 的版本变更
- 每个数据文件记录写入时的 Schema 版本
- 读取时根据当前 Schema 和文件 Schema 进行**自动映射**
- 支持操作：加列、删列、重命名、改类型、重排列序

### Q3：Iceberg 和 Hive 表的核心区别？

| 对比 | Iceberg | Hive |
|------|---------|------|
| **表格式** | 开放表格式 | 传统 Metastore |
| **Schema 变更** | 🔥 无需重写数据 | 需要重写 |
| **分区变更** | 🔥 无需迁移数据 | 需要重新分区 |
| **ACID** | 原生支持 | 需要开启 Hive ACID（性能差） |
| **Time Travel** | ✅ 原生支持 | ❌ 不支持 |
| **隐藏分区** | ✅ 查询无需感知分区 | ❌ 查询需指定分区 |
| **并发控制** | 乐观锁 | 无 |

### Q4：COW 和 MOR 的区别和选择？

- **COW**：更新时重写整个数据文件，读取快但写入慢 → **适合读多写少**
- **MOR**：写入 Delete File + 新数据文件，写入快但读取需合并 → **适合写多读少、实时场景**
- 🔥 **生产建议**：实时数仓用 MOR（v2 格式），离线分析用 COW（v1 格式）
