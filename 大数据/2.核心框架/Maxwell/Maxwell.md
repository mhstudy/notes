# Maxwell

> 🔗 **GitHub**：https://github.com/zendesk/maxwell
> 📌 **学习版本**：Maxwell 1.29.x

---

## 第1章 Maxwell 概述 ⭐

### 1.1 什么是 Maxwell

Maxwell 是一个**MySQL Binlog 增量数据同步工具**，能将 MySQL 的数据变更实时同步到 Kafka、Kinesis 等目标系统，输出 JSON 格式。

### 1.2 工作原理 🔥

```
MySQL Binlog → Maxwell（伪装为 MySQL Slave）→ Kafka/Kinesis
```

Maxwell 将自己伪装成 MySQL 的从节点（Slave），接收主节点推送的 Binlog，解析后发送到下游。

### 1.3 Maxwell vs Canal vs FlinkCDC 🔥🔥

| 对比 | Maxwell | Canal | FlinkCDC |
|:---|:---|:---|:---|
| 开发语言 | Java | Java | Java |
| 输出格式 | **JSON**（简洁） | 自有协议 | Flink DataStream |
| 输出目标 | Kafka 等 | Kafka/RocketMQ 等 | Flink 下游 |
| 全量同步 | ✅ 支持（bootstrap） | ❌ | ✅ 支持 |
| 断点续传 | ✅ | ✅ | ✅ |
| 资源占用 | **轻量** | 较重 | 依赖 Flink 集群 |
| 使用场景 | 简单增量同步 | 企业级 CDC | 实时流处理 |

---

## 第2章 配置与使用 ⭐

### 2.1 MySQL 配置

```sql
-- 开启 Binlog
-- /etc/my.cnf
-- [mysqld]
-- server-id=1
-- log-bin=mysql-bin
-- binlog_format=row        -- 必须设为 ROW 格式！
-- binlog-do-db=gmall       -- 监控的数据库

-- 创建 Maxwell 元数据库
CREATE DATABASE maxwell;
-- 授权
GRANT ALL ON maxwell.* TO 'maxwell'@'%' IDENTIFIED BY 'maxwell';
GRANT SELECT, REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'maxwell'@'%';
```

### 2.2 Maxwell 配置

```properties
# config.properties
producer=kafka
kafka.bootstrap.servers=hadoop102:9092,hadoop103:9092
kafka_topic=topic_db

# MySQL 连接
host=hadoop102
user=maxwell
password=maxwell
jdbc_options=useSSL=false&serverTimezone=Asia/Shanghai

# 过滤
filter=exclude: *.*, include: gmall.*
```

### 2.3 启动命令

```bash
# 增量同步
/opt/module/maxwell/bin/maxwell --config /opt/module/maxwell/config.properties --daemon

# 全量同步（bootstrap）🔥
/opt/module/maxwell/bin/maxwell-bootstrap --config /opt/module/maxwell/config.properties --database gmall --table user_info
```

### 2.4 输出格式

```json
{
    "database": "gmall",
    "table": "user_info",
    "type": "insert",          // insert / update / delete
    "ts": 1686815400,
    "data": {
        "id": 1001,
        "name": "张三",
        "age": 25
    },
    "old": null                 // update 时为旧值
}
```

---

## 第3章 全量同步（Bootstrap）🔥

### 3.1 Bootstrap 原理

```
1. Maxwell 向 maxwell.bootstrap 表插入一条任务记录
2. Maxwell 检测到任务后，直接 SELECT 全表数据
3. 输出 type="bootstrap-insert" 的 JSON 到 Kafka
4. 全量完成后，继续监听 Binlog 增量
```

### 3.2 Bootstrap 命令

```bash
# 同步单张表
/opt/module/maxwell/bin/maxwell-bootstrap \
    --config /opt/module/maxwell/config.properties \
    --database gmall \
    --table user_info

# 同步带条件
/opt/module/maxwell/bin/maxwell-bootstrap \
    --config /opt/module/maxwell/config.properties \
    --database gmall \
    --table order_info \
    --where "create_time >= '2023-01-01'"
```

### 3.3 Bootstrap 输出格式

```json
// 开始标记
{"database":"gmall","table":"user_info","type":"bootstrap-start","ts":1686815400,"data":{}}

// 数据记录
{"database":"gmall","table":"user_info","type":"bootstrap-insert","ts":1686815401,
 "data":{"id":1001,"name":"张三","age":25}}

// 完成标记
{"database":"gmall","table":"user_info","type":"bootstrap-complete","ts":1686815500,"data":{}}
```

---

## 第4章 断点续传 ⭐

### 4.1 原理

Maxwell 将消费的 Binlog 位点信息存储在 **maxwell.positions** 表中：

```sql
SELECT * FROM maxwell.positions;
-- server_id | binlog_file    | binlog_position | last_heartbeat_read
-- 1         | mysql-bin.000003 | 15462           | 1686815400
```

重启后自动从上次记录的位点继续消费。

### 4.2 位点管理

```bash
# 查看当前位点
mysql -e "SELECT * FROM maxwell.positions"

# 重置位点（重新消费）
mysql -e "DELETE FROM maxwell.positions"

# 指定位点启动
/opt/module/maxwell/bin/maxwell \
    --config config.properties \
    --init_position=mysql-bin.000003:15462
```

---

## 第5章 生产部署 ⭐

### 5.1 启停脚本

```bash
#!/bin/bash
# maxwell.sh
MAXWELL_HOME=/opt/module/maxwell

case $1 in
"start")
    echo "========== 启动 Maxwell =========="
    nohup $MAXWELL_HOME/bin/maxwell \
        --config $MAXWELL_HOME/config.properties \
        --daemon > /dev/null 2>&1 &
    ;;
"stop")
    echo "========== 停止 Maxwell =========="
    ps -ef | grep maxwell | grep -v grep | awk '{print $2}' | xargs kill -9
    ;;
"restart")
    $0 stop
    sleep 2
    $0 start
    ;;
esac
```

### 5.2 Kafka Topic 分区策略

```properties
# 按表名分区（保证同一表的数据有序）
producer_partition_by=table

# 按主键分区（保证同一主键的数据有序）
producer_partition_by=primary_key

# 按库名分区
producer_partition_by=database
```

### 5.3 常见问题

| 问题 | 解决方案 |
|:---|:---|
| Maxwell 连接 MySQL 失败 | 检查 Binlog 是否开启、格式是否为 ROW |
| 数据延迟大 | 检查 Kafka 消费是否积压，增加消费者 |
| Bootstrap 卡住 | 检查 maxwell.bootstrap 表状态 |
| 位点丢失 | 检查 MySQL Binlog 过期配置 `expire_logs_days` |

---

## 第6章 面试题 🔥🔥

### Q1：Maxwell 的原理？
> 伪装为 MySQL Slave，接收 Binlog，解析为 JSON 发送到 Kafka。

### Q2：为什么 Binlog 必须是 ROW 格式？
> ROW 格式记录每行数据的变化，而 STATEMENT 只记录 SQL 语句，无法精确还原数据变更。MIXED 模式不确定性太高。

### Q3：Maxwell 如何实现全量同步？
> 通过 Bootstrap 机制：向 maxwell.bootstrap 表写入任务 → SELECT 全表数据 → 输出 `bootstrap-insert` 类型 JSON → 完成后继续 Binlog 增量。

### Q4：Maxwell 如何保证断点续传？
> 将 Binlog 位点（文件名 + 偏移量）持久化到 maxwell.positions 表，重启后从上次位点继续消费。

### Q5：Maxwell 和 Canal 的区别？
> | 对比 | Maxwell | Canal |
> |:---|:---|:---|
> | 输出格式 | **JSON 简洁** | 自有 protobuf 格式 |
> | 全量同步 | ✅ Bootstrap | ❌ 不支持 |
> | 资源占用 | 轻量单进程 | 较重（Server + Client） |
> | HA | 依赖外部 | 内置 HA |
> | 适用场景 | 中小规模、快速搭建 | 大规模、企业级 |
