# DataX

> 🔗 **GitHub**：https://github.com/alibaba/DataX
> 📌 **学习版本**：DataX 3.x（阿里开源）

---

## 第1章 DataX 概述 ⭐

### 1.1 什么是 DataX

DataX 是阿里开源的**离线数据同步工具**，实现各种异构数据源之间的高效数据同步。

### 1.2 架构原理 🔥

```
┌──────────┐    ┌─────────────────────────┐    ┌──────────┐
│  Reader  │───▶│  Framework (Channel)    │───▶│  Writer  │
│ (数据源)  │    │  传输 + 缓冲 + 流控      │    │ (目标源)  │
└──────────┘    └─────────────────────────┘    └──────────┘
```

采用 **Framework + Plugin** 架构：
- **Reader**：数据读取插件（MySQL、HDFS、Oracle...）
- **Writer**：数据写入插件（HDFS、MySQL、HBase...）
- **Channel**：数据传输通道，负责缓冲和限速

### 1.3 DataX vs Sqoop 🔥

| 对比 | DataX | Sqoop |
|:---|:---|:---|
| 架构 | 单机多线程 | MR 分布式 |
| 数据源 | **丰富**（20+） | 主要 RDBMS↔HDFS |
| 速度 | 单机高效 | 分布式大规模 |
| 维护 | 阿里维护，社区活跃 | 已停止维护 |
| 配置 | JSON 配置文件 | 命令行参数 |

---

## 第2章 使用案例 🔥

### 2.1 MySQL → HDFS

```json
{
    "job": {
        "setting": {
            "speed": { "channel": 3 },
            "errorLimit": { "record": 0, "percentage": 0.02 }
        },
        "content": [{
            "reader": {
                "name": "mysqlreader",
                "parameter": {
                    "username": "root",
                    "password": "000000",
                    "column": ["id", "name", "age", "create_time"],
                    "splitPk": "id",
                    "connection": [{
                        "table": ["user_info"],
                        "jdbcUrl": ["jdbc:mysql://hadoop102:3306/gmall"]
                    }]
                }
            },
            "writer": {
                "name": "hdfswriter",
                "parameter": {
                    "defaultFS": "hdfs://hadoop102:8020",
                    "fileType": "orc",
                    "path": "/origin_data/gmall/db/user_info_full/${dt}",
                    "fileName": "user_info",
                    "column": [
                        {"name": "id", "type": "bigint"},
                        {"name": "name", "type": "string"},
                        {"name": "age", "type": "int"},
                        {"name": "create_time", "type": "string"}
                    ],
                    "writeMode": "append",
                    "fieldDelimiter": "\t",
                    "compress": "snappy"
                }
            }
        }]
    }
}
```

```bash
# 执行
python /opt/module/datax/bin/datax.py /opt/module/datax/job/mysql2hdfs.json -p "-Ddt=2023-06-15"
```

### 2.2 HDFS → MySQL

```json
{
    "job": {
        "content": [{
            "reader": {
                "name": "hdfsreader",
                "parameter": {
                    "path": "/origin_data/export/ads_result/",
                    "defaultFS": "hdfs://hadoop102:8020",
                    "fileType": "orc",
                    "column": [
                        {"index": 0, "type": "string"},
                        {"index": 1, "type": "long"}
                    ]
                }
            },
            "writer": {
                "name": "mysqlwriter",
                "parameter": {
                    "writeMode": "replace",
                    "username": "root",
                    "password": "000000",
                    "column": ["dt", "count"],
                    "connection": [{
                        "jdbcUrl": "jdbc:mysql://hadoop102:3306/gmall_report",
                        "table": ["ads_daily_count"]
                    }]
                }
            }
        }],
        "setting": { "speed": { "channel": 1 } }
    }
}
```

---

## 第3章 常用 Reader/Writer ⭐

| Reader 插件 | 数据源 | Writer 插件 | 目标源 |
|:---|:---|:---|:---|
| mysqlreader | MySQL | mysqlwriter | MySQL |
| hdfsreader | HDFS | hdfswriter | HDFS |
| oraclereader | Oracle | oraclewriter | Oracle |
| postgresqlreader | PostgreSQL | postgresqlwriter | PostgreSQL |
| mongodbreader | MongoDB | mongodbwriter | MongoDB |
| hbase11xreader | HBase | hbase11xwriter | HBase |
| streamreader | 测试数据 | streamwriter | 控制台输出 |

---

## 第4章 传输调优 🔥🔥

### 4.1 速度优化

```json
{
    "job": {
        "setting": {
            "speed": {
                "channel": 5,              // 并发通道数（核心参数）🔥
                "byte": 5242880,           // 每秒限速（5MB/s），0 不限速
                "record": 100000           // 每秒限制记录数
            },
            "errorLimit": {
                "record": 0,               // 允许脏数据条数
                "percentage": 0.02         // 允许脏数据比例
            }
        }
    }
}
```

### 4.2 JVM 调优

```bash
# datax.py 中调整 JVM 参数
python datax.py --jvm="-Xms4G -Xmx4G" /opt/module/datax/job/mysql2hdfs.json

# 或修改 datax.py 中的默认配置
DEFAULT_JVM = "-Xms1g -Xmx1g -XX:+HeapDumpOnOutOfMemoryError"
```

### 4.3 Channel 数量建议

| 数据量 | Channel 数 | 说明 |
|:---|:---|:---|
| < 100 万行 | 1~3 | 小数据量，无需过多并发 |
| 100~1000 万行 | 3~5 | 常规场景 |
| > 1000 万行 | 5~10 | 大数据量，注意内存和数据库连接数 |

> ⚠️ Channel 并非越大越好！过多会导致数据库连接耗尽、内存不足。

---

## 第5章 DataX 生产实践 ⭐

### 5.1 批量生成配置脚本

```python
#!/usr/bin/env python
# gen_datax_config.py —— 批量生成 MySQL → HDFS 的 DataX 配置
import json, os

TABLES = ["user_info", "order_info", "sku_info", "base_province"]
MYSQL_URL = "jdbc:mysql://hadoop102:3306/gmall"
HDFS_PATH = "/origin_data/gmall/db"

for table in TABLES:
    config = {
        "job": {
            "setting": {"speed": {"channel": 3}},
            "content": [{
                "reader": {
                    "name": "mysqlreader",
                    "parameter": {
                        "username": "root",
                        "password": "000000",
                        "column": ["*"],
                        "connection": [{"table": [table], "jdbcUrl": [MYSQL_URL]}]
                    }
                },
                "writer": {
                    "name": "hdfswriter",
                    "parameter": {
                        "defaultFS": "hdfs://hadoop102:8020",
                        "fileType": "orc",
                        "path": f"{HDFS_PATH}/{table}_full/${{dt}}",
                        "fileName": table,
                        "writeMode": "append",
                        "compress": "snappy"
                    }
                }
            }]
        }
    }
    with open(f"/opt/module/datax/job/{table}.json", "w") as f:
        json.dump(config, f, indent=4, ensure_ascii=False)
    print(f"Generated: {table}.json")
```

### 5.2 DataX 执行脚本

```bash
#!/bin/bash
# mysql_to_hdfs.sh —— 全量同步脚本
DATAX_HOME=/opt/module/datax
DT=$1  # 传入日期参数

[ -z "$DT" ] && DT=$(date -d '-1 day' +%F)

TABLES="user_info order_info sku_info base_province"

for table in $TABLES; do
    echo "========== 同步 $table ($DT) =========="
    python $DATAX_HOME/bin/datax.py \
        $DATAX_HOME/job/${table}.json \
        -p "-Ddt=$DT"
done
echo "========== 全部同步完成 =========="
```

### 5.3 数据校验

```bash
# 同步后对比源端和目标端行数
mysql -uroot -p000000 -e "SELECT COUNT(*) FROM gmall.user_info"
hdfs dfs -cat /origin_data/gmall/db/user_info_full/2023-06-15/* | wc -l
```

---

## 第6章 面试题 🔥🔥

### Q1：DataX 如何提升同步速度？
> 1. 增加 `channel` 数量（并行度）
> 2. 设置 `splitPk`（Reader 端按主键分片并行读）
> 3. 调整 JVM 内存：`-Xms` / `-Xmx`
> 4. 使用高效文件格式（ORC + Snappy 压缩）
> 5. 去掉限速配置（byte / record 设为 0）

### Q2：DataX 和 Sqoop 怎么选？
> DataX 功能更丰富、配置更灵活，是当前主流选择。Sqoop 已停止维护，新项目推荐 DataX。

### Q3：DataX 的架构原理？
> **Framework + Plugin** 架构。Reader 从数据源读取，经过 Channel 缓冲传输（内存中的 Record 队列），由 Writer 写入目标。支持限速和脏数据管理。

### Q4：DataX Channel 设置多大合适？
> 取决于数据量和资源限制。小表 1~3，大表 5~10。注意数据库连接数限制和 JVM 内存。过多 Channel 可能导致源端压力过大。

### Q5：DataX 如何处理脏数据？
> 通过 `errorLimit` 配置：`record` 设置允许的脏数据条数，`percentage` 设置允许的脏数据比例。超过阈值任务报错终止。脏数据会记录到日志中。
