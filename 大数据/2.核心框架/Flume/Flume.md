# Flume

> 🔗 **官方网站**：https://flume.apache.org/
> 📖 **官方文档**：https://flume.apache.org/FlumeUserGuide.html
> 📌 **学习版本**：Flume 1.11.x

---

## 第1章 Flume 概述 ⭐

### 1.1 什么是 Flume

Flume 是一个高可用、高可靠、分布式的**海量日志采集、聚合和传输系统**。

### 1.2 核心架构 🔥🔥

![Flume架构](https://flume.apache.org/_images/DevGuide_image00.png ':size=600')

| 组件 | 说明 |
|:---|:---|
| **Agent** | JVM 进程，包含 Source、Channel、Sink |
| **Source** | 数据采集源（接收数据） |
| **Channel** | 数据缓冲通道（Source→Channel→Sink） |
| **Sink** | 数据输出目标（输出数据） |

### 1.3 事务机制 🔥

| 事务 | 说明 |
|:---|:---|
| **Put 事务** | Source → Channel（写入通道时的事务保证） |
| **Take 事务** | Channel → Sink（从通道读取时的事务保证） |

> 💡 **面试问法**：Flume 采集数据会丢失吗？
> **答**：使用 **FileChannel** 时不会丢失（磁盘持久化），使用 MemoryChannel 可能丢失（内存）。Channel 的 Put/Take 事务机制保证数据可靠传输。

---

## 第2章 常用组件 🔥

### 2.1 Source 类型

| Source | 说明 | 适用场景 |
|:---|:---|:---|
| **Taildir Source** 🔥 | 监控目录下文件变化，**支持断点续传** | 生产首选 |
| Exec Source | 执行 Linux 命令（如 `tail -f`） | 不推荐（可能丢数据）|
| Spooling Dir Source | 监控目录新文件 | 一次性文件 |
| Avro Source | 接收 Avro 数据 | Agent 级联 |
| Kafka Source | 从 Kafka 消费 | Kafka 集成 |

### 2.2 Channel 类型

| Channel | 说明 | 速度 | 可靠性 |
|:---|:---|:---|:---|
| **Memory Channel** | 内存缓冲 | 快 | 低（宕机丢数据）|
| **File Channel** 🔥 | 磁盘缓冲 | 慢 | **高**（生产推荐）|
| **Kafka Channel** 🔥 | Kafka 作为 Channel | 快 | 高 |

### 2.3 Sink 类型

| Sink | 说明 | 适用场景 |
|:---|:---|:---|
| **HDFS Sink** 🔥 | 写入 HDFS | 离线数据采集 |
| **Kafka Sink** 🔥 | 写入 Kafka | 实时数据管道 |
| Avro Sink | 发送到下游 Agent | Agent 级联 |
| Logger Sink | 打印日志 | 测试调试 |

---

## 第3章 配置案例 🔥

### 3.1 实时监控文件到 HDFS

```properties
# flume-file-hdfs.conf
# Agent 命名
a1.sources = r1
a1.channels = c1
a1.sinks = k1

# Source: Taildir（断点续传）
a1.sources.r1.type = TAILDIR
a1.sources.r1.filegroups = f1
a1.sources.r1.filegroups.f1 = /opt/module/data/logs/app.*
a1.sources.r1.positionFile = /opt/module/flume/taildir_position.json

# Channel: File（可靠）
a1.channels.c1.type = file
a1.channels.c1.checkpointDir = /opt/module/flume/checkpoint
a1.channels.c1.dataDirs = /opt/module/flume/data

# Sink: HDFS
a1.sinks.k1.type = hdfs
a1.sinks.k1.hdfs.path = hdfs://hadoop102:8020/flume/%Y%m%d/%H
a1.sinks.k1.hdfs.filePrefix = logs-
a1.sinks.k1.hdfs.round = true
a1.sinks.k1.hdfs.roundValue = 1
a1.sinks.k1.hdfs.roundUnit = hour
a1.sinks.k1.hdfs.rollInterval = 30      # 30秒滚动一次
a1.sinks.k1.hdfs.rollSize = 134217700   # 128MB 滚动
a1.sinks.k1.hdfs.rollCount = 0          # 不按条数滚动
a1.sinks.k1.hdfs.fileType = CompressedStream
a1.sinks.k1.hdfs.codeC = gzip

# 组装
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

### 3.2 监控日志到 Kafka

```properties
# flume-file-kafka.conf
a1.sources = r1
a1.channels = c1
a1.sinks = k1

# Source: Taildir
a1.sources.r1.type = TAILDIR
a1.sources.r1.filegroups = f1
a1.sources.r1.filegroups.f1 = /opt/module/data/logs/app.*

# Channel: Kafka（高性能 + 可靠）
a1.channels.c1.type = org.apache.flume.channel.kafka.KafkaChannel
a1.channels.c1.kafka.bootstrap.servers = hadoop102:9092,hadoop103:9092
a1.channels.c1.kafka.topic = topic_log
a1.channels.c1.parseAsFlumeEvent = false

# 组装（使用 Kafka Channel 时无需 Sink）
a1.sources.r1.channels = c1
```

### 3.3 自定义拦截器 🔥

```java
import org.apache.flume.Context;
import org.apache.flume.Event;
import org.apache.flume.interceptor.Interceptor;
import java.nio.charset.StandardCharsets;
import java.util.*;

public class TimestampInterceptor implements Interceptor {
    
    @Override
    public void initialize() {}

    @Override
    public Event intercept(Event event) {
        // 获取 body 中的 JSON，提取 ts 字段作为 Header
        String body = new String(event.getBody(), StandardCharsets.UTF_8);
        // 假设 JSON 中有 ts 字段
        Map<String, String> headers = event.getHeaders();
        headers.put("timestamp", String.valueOf(System.currentTimeMillis()));
        return event;
    }

    @Override
    public List<Event> intercept(List<Event> events) {
        List<Event> result = new ArrayList<>();
        for (Event event : events) {
            Event e = intercept(event);
            if (e != null) result.add(e);
        }
        return result;
    }

    @Override
    public void close() {}

    public static class Builder implements Interceptor.Builder {
        @Override
        public Interceptor build() { return new TimestampInterceptor(); }
        @Override
        public void configure(Context context) {}
    }
}
```

---

## 第4章 高频面试题 🔥🔥🔥

### Q1：Flume 的 Source、Channel、Sink 分别有哪些常用类型？
> Source：Taildir（生产首选，支持断点续传）、Exec、Kafka Source
> Channel：FileChannel（可靠）、MemoryChannel（快）、KafkaChannel（推荐）
> Sink：HDFS Sink、Kafka Sink、Avro Sink

### Q2：Flume 采集数据会丢失吗？
> 使用 FileChannel 或 KafkaChannel 不会丢失。MemoryChannel 在 Agent 宕机时可能丢失。

### Q3：Flume 如何实现断点续传？
> 使用 **TailDir Source**，它会将读取位置记录在 `positionFile` 中，重启后从断点继续读取。

### Q4：Flume 如何解决 HDFS 小文件问题？
> 合理配置 Sink 的滚动参数：`rollInterval`（时间）、`rollSize`（大小）、`rollCount`（条数），建议设置 128MB 滚动。
