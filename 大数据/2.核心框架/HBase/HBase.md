# HBase

> 🔗 **官方网站**：https://hbase.apache.org/
> 📖 **官方文档**：https://hbase.apache.org/book.html
> 📌 **学习版本**：HBase 2.4.x on Hadoop 3.3.x

---

## 第1章 HBase 概述 ⭐

### 1.1 什么是 HBase

HBase 是一个**分布式、面向列的 NoSQL 数据库**，运行在 HDFS 之上，适合存储**海量稀疏数据**，提供实时随机读写能力。

**特点**：高可靠、高性能、面向列、可伸缩、实时读写

### 1.2 HBase 架构 🔥🔥

![HBase架构](https://hbase.apache.org/images/hbase_architecture.png ':size=700')

| 组件 | 职责 |
|:---|:---|
| **HMaster** | 管理 RegionServer，负责 Region 分配、DDL 操作、负载均衡 |
| **RegionServer** | 管理 Region，负责读写数据，执行 Flush/Compact/Split |
| **Region** | 表按 RowKey 范围划分的数据分片 |
| **Store** | 一个列族对应一个 Store |
| **MemStore** | 写缓冲（内存），默认 128MB |
| **StoreFile/HFile** | 数据持久化文件（HDFS 上） |
| **WAL（HLog）** | 预写日志，保证数据不丢失 |
| **ZooKeeper** | Master 选举、RegionServer 监控、元数据管理 |

### 1.3 数据模型 🔥🔥

| 概念 | 说明 |
|:---|:---|
| **NameSpace** | 命名空间（类似数据库），默认 `default` 和 `hbase` |
| **Table** | 表 |
| **Row** | 行，按 **RowKey** 字典序排列 |
| **Column Family** | 列族，建表时指定，数量不宜过多（1~3个）|
| **Column Qualifier** | 列限定符，列族下的具体列 |
| **Cell** | 单元格 = RowKey + ColumnFamily + Qualifier + Timestamp |
| **TimeStamp** | 时间戳，每条数据带版本号 |

```
RowKey      |  info:name  |  info:age  |  detail:addr
------------|-------------|------------|------------------
row001      |  张三       |  25        |  北京
row002      |  李四       |  30        |  上海
```

---

## 第2章 Shell 操作 ⭐

```bash
# 进入 HBase Shell
hbase shell

# 命名空间操作
create_namespace 'bigdata'
list_namespace

# 建表
create 'bigdata:student', 'info', 'detail'
create 'student', {NAME => 'info', VERSIONS => 3}, {NAME => 'detail'}

# 查看表
list
describe 'student'

# 增
put 'student', '1001', 'info:name', '张三'
put 'student', '1001', 'info:age', '25'
put 'student', '1001', 'detail:addr', '北京'

# 查
get 'student', '1001'
get 'student', '1001', 'info:name'
get 'student', '1001', {COLUMN => 'info:name', VERSIONS => 3}

# 扫描
scan 'student'
scan 'student', {STARTROW => '1001', STOPROW => '1003'}
scan 'student', {LIMIT => 5}

# 删
delete 'student', '1001', 'info:age'
deleteall 'student', '1001'
truncate 'student'

# 表管理
disable 'student'
drop 'student'
```

---

## 第3章 Java API 🔥

```java
import org.apache.hadoop.hbase.*;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.util.Bytes;
import org.apache.hadoop.conf.Configuration;

public class HBaseAPI {
    private static Connection connection;

    static {
        try {
            Configuration conf = HBaseConfiguration.create();
            conf.set("hbase.zookeeper.quorum", "hadoop102,hadoop103,hadoop104");
            connection = ConnectionFactory.createConnection(conf);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // 创建表
    public static void createTable(String tableName, String... columnFamilies) throws Exception {
        Admin admin = connection.getAdmin();
        if (admin.tableExists(TableName.valueOf(tableName))) {
            System.out.println("表已存在");
            return;
        }
        TableDescriptorBuilder builder = TableDescriptorBuilder.newBuilder(TableName.valueOf(tableName));
        for (String cf : columnFamilies) {
            builder.setColumnFamily(ColumnFamilyDescriptorBuilder.of(cf));
        }
        admin.createTable(builder.build());
        admin.close();
    }

    // 写入数据
    public static void putData(String tableName, String rowKey, String cf, String col, String val) throws Exception {
        Table table = connection.getTable(TableName.valueOf(tableName));
        Put put = new Put(Bytes.toBytes(rowKey));
        put.addColumn(Bytes.toBytes(cf), Bytes.toBytes(col), Bytes.toBytes(val));
        table.put(put);
        table.close();
    }

    // 读取数据
    public static void getData(String tableName, String rowKey) throws Exception {
        Table table = connection.getTable(TableName.valueOf(tableName));
        Get get = new Get(Bytes.toBytes(rowKey));
        Result result = table.get(get);
        for (Cell cell : result.rawCells()) {
            System.out.println(
                "RowKey: " + Bytes.toString(CellUtil.cloneRow(cell)) +
                ", CF: " + Bytes.toString(CellUtil.cloneFamily(cell)) +
                ", Col: " + Bytes.toString(CellUtil.cloneQualifier(cell)) +
                ", Val: " + Bytes.toString(CellUtil.cloneValue(cell))
            );
        }
        table.close();
    }

    // 扫描
    public static void scanData(String tableName, String startRow, String stopRow) throws Exception {
        Table table = connection.getTable(TableName.valueOf(tableName));
        Scan scan = new Scan();
        scan.withStartRow(Bytes.toBytes(startRow));
        scan.withStopRow(Bytes.toBytes(stopRow));
        ResultScanner scanner = table.getScanner(scan);
        for (Result result : scanner) {
            for (Cell cell : result.rawCells()) {
                System.out.println(Bytes.toString(CellUtil.cloneRow(cell)) + " => " +
                    Bytes.toString(CellUtil.cloneValue(cell)));
            }
        }
        scanner.close();
        table.close();
    }

    // 删除
    public static void deleteData(String tableName, String rowKey, String cf, String col) throws Exception {
        Table table = connection.getTable(TableName.valueOf(tableName));
        Delete delete = new Delete(Bytes.toBytes(rowKey));
        delete.addColumn(Bytes.toBytes(cf), Bytes.toBytes(col));
        table.delete(delete);
        table.close();
    }
}
```

---

## 第4章 读写流程 🔥🔥🔥

### 4.1 写流程 🔥🔥

```
Client → ZooKeeper（找到 Meta 表位置）
       → Meta 表（找到目标 RegionServer）
       → RegionServer
           → WAL（先写预写日志）
           → MemStore（写内存缓冲）
           → 返回成功

MemStore 满后 → Flush 为 StoreFile（HFile）
StoreFile 多了 → Compaction（合并）
Region 大了   → Split（分裂）
```

> 💡 **面试必背**：写流程 = Client → ZK → Meta → RegionServer → WAL → MemStore → 返回ACK

### 4.2 读流程 🔥🔥

```
Client → ZooKeeper（找到 Meta 表位置）
       → Meta 表（找到目标 RegionServer）
       → RegionServer
           → Block Cache（先查读缓存）
           → MemStore（查内存）
           → StoreFile（查磁盘 HFile）
           → 合并结果，返回最新版本
```

### 4.3 Flush 机制 ⭐

| 触发条件 | 参数 |
|:---|:---|
| 单个 MemStore 达到 128MB | `hbase.hregion.memstore.flush.size` |
| RegionServer 所有 MemStore 达到堆内存 40% | `hbase.regionserver.global.memstore.size` |
| WAL 文件数超过阈值 | `hbase.regionserver.maxlogs` |
| 定期刷写 | `hbase.regionserver.optionalcacheflushinterval`（默认1小时）|

### 4.4 Compaction 合并 🔥

| 类型 | 说明 |
|:---|:---|
| **Minor Compaction** | 选取几个小的 HFile 合并，**不删除数据** |
| **Major Compaction** | 合并所有 HFile，**删除过期数据、标记删除的数据** |

---

## 第5章 RowKey 设计 🔥🔥🔥

> **RowKey 设计是 HBase 面试最高频考点！**

### 5.1 设计三大原则

| 原则 | 说明 | 目的 |
|:---|:---|:---|
| **唯一性** | RowKey 必须唯一 | 数据不覆盖 |
| **长度原则** | 建议 10~100 字节 | 节省存储、提高效率 |
| **散列原则** | 避免热点问题 | 数据均匀分布 |

### 5.2 热点问题解决方案 🔥

| 方案 | 做法 | 适用场景 |
|:---|:---|:---|
| **加盐（Salting）** | RowKey 前缀加随机数 | 写多读少 |
| **Hash** | `hash(原始key) + 原始key` | 通用 |
| **反转** | 手机号/时间戳反转 | 按时间递增的场景 |
| **预分区** | 建表时指定分区范围 | 已知数据分布 |

```java
// 反转示例：手机号反转
String phone = "13812345678";
String reversedPhone = new StringBuilder(phone).reverse().toString();
// "87654321831" → 散列到不同Region

// Hash 示例
String rowKey = DigestUtils.md5Hex(phone).substring(0, 8) + "_" + phone;

// 预分区建表
create 'user', 'info', SPLITS => ['10', '20', '30', '40', '50', '60', '70', '80', '90']
```

### 5.3 实战案例：电商用户行为表

```
需求：存储用户浏览记录，查询维度：用户ID + 时间范围

RowKey 设计：
hash(userId)[0:4] + "_" + userId + "_" + (Long.MAX_VALUE - timestamp)

示例：a3b2_user001_9999999999999986322

优点：
1. hash前缀 → 散列，避免热点
2. userId → 同一用户数据在相邻Region
3. 反转时间戳 → 最新数据排在前面（scan 更高效）
```

---

## 第6章 高频面试题 🔥🔥🔥

### Q1：HBase 的读写流程？
> 见第4章详解。核心要记住：写→WAL→MemStore，读→BlockCache→MemStore→HFile

### Q2：RowKey 如何设计？如何避免热点？
> 三大原则：唯一性、长度适中、散列性。解决热点：加盐、Hash、反转、预分区。

### Q3：HBase 和 Hive 的区别？
> | 对比 | HBase | Hive |
> |:---|:---|:---|
> | 类型 | NoSQL 数据库 | 数据仓库 |
> | 延迟 | **毫秒级** | 分钟级 |
> | 操作 | CRUD | 批量分析 |
> | 存储 | KV 列式 | 行式/列式文件 |
> | 场景 | 实时读写 | 离线分析 |

### Q4：Minor Compaction 和 Major Compaction 的区别？
> - Minor：合并少量 HFile，不删除数据，IO 开销小
> - Major：合并所有 HFile，删除过期/标记删除的数据，IO 开销大，建议业务低峰期执行

### Q5：HBase 适合什么场景？
> 1. 海量数据存储（十亿级行）
> 2. 实时随机读写
> 3. 稀疏数据
> 4. 典型：用户画像、消息存储、时序数据、日志存储
