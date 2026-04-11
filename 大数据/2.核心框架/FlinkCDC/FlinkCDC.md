# Flink CDC

> 🔗 **官方文档**：https://nightlies.apache.org/flink/flink-cdc-docs-stable/
> 📌 **学习版本**：Flink CDC 2.x / 3.x

---

## 第1章 FlinkCDC 概述 ⭐

### 1.1 什么是 CDC

CDC（Change Data Capture）变更数据捕获，是一种通过监控数据源变化来实现数据同步的技术。

### 1.2 FlinkCDC 特点 🔥

- 基于 Flink 实现，天然支持**流批一体**
- 支持**全量 + 增量**无缝衔接
- **无锁读取**（2.x 开始），不影响源数据库性能 🔥
- 支持 MySQL、PostgreSQL、MongoDB、Oracle 等

### 1.3 全量+增量无锁读取原理 🔥🔥

```
阶段1: Snapshot（全量读取）
  - 使用快照读，不加全局锁
  - 按主键范围切分，多并行度读取

阶段2: Binlog（增量读取）
  - 从快照结束的 Binlog 位点继续消费
  - 通过比对确保 exactly-once

无缝切换，保证数据不丢不重！
```

---

## 第2章 DataStream API 🔥

```java
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import com.ververica.cdc.connectors.mysql.source.MySqlSource;
import com.ververica.cdc.debezium.JsonDebeziumDeserializationSchema;

public class FlinkCDCExample {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        env.enableCheckpointing(5000);

        MySqlSource<String> mySqlSource = MySqlSource.<String>builder()
                .hostname("hadoop102")
                .port(3306)
                .databaseList("gmall")
                .tableList("gmall.user_info", "gmall.order_info")
                .username("root")
                .password("000000")
                .deserializer(new JsonDebeziumDeserializationSchema())
                .startupOptions(StartupOptions.initial())  // 全量+增量
                .build();

        env.fromSource(mySqlSource, WatermarkStrategy.noWatermarks(), "MySQL Source")
           .print();

        env.execute("Flink CDC Job");
    }
}
```

## 第3章 FlinkSQL 方式 🔥

```sql
-- 创建 CDC Source 表
CREATE TABLE user_info (
    id INT,
    name STRING,
    age INT,
    PRIMARY KEY (id) NOT ENFORCED
) WITH (
    'connector' = 'mysql-cdc',
    'hostname' = 'hadoop102',
    'port' = '3306',
    'username' = 'root',
    'password' = '000000',
    'database-name' = 'gmall',
    'table-name' = 'user_info'
);

-- 实时查询变更
SELECT * FROM user_info;

-- 写入 Kafka
CREATE TABLE user_info_kafka (
    id INT,
    name STRING,
    age INT,
    PRIMARY KEY (id) NOT ENFORCED
) WITH (
    'connector' = 'upsert-kafka',
    'topic' = 'user_info',
    'properties.bootstrap.servers' = 'hadoop102:9092',
    'key.format' = 'json',
    'value.format' = 'json'
);

INSERT INTO user_info_kafka SELECT * FROM user_info;
```

---

## 第4章 自定义反序列化 ⭐

```java
// 自定义 JSON 输出格式
public class CustomDeserializationSchema implements DebeziumDeserializationSchema<String> {
    @Override
    public void deserialize(SourceRecord record, Collector<String> out) {
        Struct value = (Struct) record.value();
        Struct source = value.getStruct("source");

        JSONObject result = new JSONObject();
        result.put("database", source.getString("db"));
        result.put("table", source.getString("table"));

        // 操作类型
        Envelope.Operation op = Envelope.operationFor(record);
        String type = op == Envelope.Operation.CREATE ? "insert"
                    : op == Envelope.Operation.UPDATE ? "update"
                    : "delete";
        result.put("type", type);

        // 变更后数据
        Struct after = value.getStruct("after");
        if (after != null) {
            JSONObject data = new JSONObject();
            for (Field field : after.schema().fields()) {
                data.put(field.name(), after.get(field));
            }
            result.put("data", data);
        }

        out.collect(result.toJSONString());
    }

    @Override
    public TypeInformation<String> getProducedType() {
        return BasicTypeInfo.STRING_TYPE_INFO;
    }
}
```

---

## 第5章 Flink CDC 3.x Pipeline 模式 🔥🔥

### 5.1 什么是 Pipeline

Flink CDC 3.x 引入了 **YAML Pipeline** 模式，无需编写代码，通过配置文件即可完成**整库同步**。

### 5.2 Pipeline YAML 配置

```yaml
# mysql-to-doris.yaml
source:
  type: mysql
  hostname: hadoop102
  port: 3306
  username: root
  password: "000000"
  tables: gmall.\.*            # 正则匹配整库
  server-id: 5400-5404

sink:
  type: doris
  fenodes: hadoop102:8030
  username: root
  password: ""
  table.create.properties.light_schema_change: true
  table.create.properties.replication_num: 1

pipeline:
  name: MySQL to Doris Pipeline
  parallelism: 4
```

### 5.3 Pipeline vs DataStream 对比

| 对比 | Pipeline 模式 | DataStream 模式 |
|:---|:---|:---|
| 使用方式 | YAML 配置，零代码 | 编写 Java 代码 |
| 同步粒度 | **整库同步**（正则匹配） | 指定表 |
| Schema 变更 | **自动同步 DDL** 🔥 | 需自行处理 |
| 灵活性 | 标准化场景 | 高度灵活 |
| 适用场景 | 数据入仓/入湖 | 复杂 ETL 逻辑 |

```bash
# 提交 Pipeline 任务
bin/flink-cdc.sh mysql-to-doris.yaml
```

---

## 第6章 生产最佳实践 ⭐

### 6.1 Checkpoint 配置

```java
env.enableCheckpointing(60000);  // 60s
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(30000);
env.getCheckpointConfig().setCheckpointTimeout(600000);
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);
env.getCheckpointConfig().setExternalizedCheckpointCleanup(
    ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
```

### 6.2 常见问题与解决

| 问题 | 原因 | 解决方案 |
|:---|:---|:---|
| 全量阶段 OOM | 大表数据量过大 | 调大 TaskManager 内存、增加并行度 |
| Binlog 位点丢失 | MySQL Binlog 被清理 | 设置合理的 `expire_logs_days` |
| 数据重复 | Checkpoint 未正确恢复 | 确保 Sink 端支持幂等写入 |
| 延迟高 | 下游写入慢 | 批量写入、异步 IO |

---

## 第7章 面试题 🔥🔥

### Q1：FlinkCDC 2.x 的无锁读取原理？
> 通过**快照分片 + Binlog 比对**实现。全量阶段按主键范围切分，多并行度读取；增量阶段从 Binlog 继续消费。通过 Checkpoint 保证 exactly-once。

### Q2：FlinkCDC 和 Maxwell/Canal 的区别？
> FlinkCDC 直接集成在 Flink 中，支持全量+增量无缝切换，无需额外中间件。Maxwell/Canal 需要独立部署，数据先写 Kafka 再由 Flink 消费。

### Q3：FlinkCDC 的 StartupOptions 有哪些？
> - `initial()`：全量 + 增量
> - `earliest()`：从最早的 Binlog 开始
> - `latest()`：从最新的 Binlog 开始
> - `specificOffset()`：从指定位点开始
> - `timestamp()`：从指定时间戳开始

### Q4：Flink CDC 3.x 的 Pipeline 模式有什么优势？
> 零代码通过 YAML 配置实现整库同步，自动同步 Schema 变更（DDL），支持正则匹配表，适合数据入仓入湖场景。

### Q5：FlinkCDC 如何保证 Exactly-Once？
> 全量阶段：快照分片 + Binlog 位点记录到 Checkpoint。增量阶段：从 Checkpoint 恢复 Binlog 位点继续消费。配合支持幂等写入或两阶段提交的 Sink 端实现端到端 Exactly-Once。

### Q6：FlinkCDC 全量同步大表 OOM 怎么办？
> 1. 增加 TaskManager 内存
> 2. 增加 Source 并行度（增加 server-id 范围）
> 3. 使用 `scan.incremental.snapshot.chunk.size` 减小分片大小
> 4. 开启 Checkpoint 避免失败全部重来
