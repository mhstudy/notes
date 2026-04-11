# Hive

> 🔗 **官方网站**：https://hive.apache.org/
> 📖 **官方文档**：https://cwiki.apache.org/confluence/display/Hive
> 📌 **学习版本**：Hive 3.1.3 on Hadoop 3.3.x

---

## 第1章 Hive 概述 ⭐

### 1.1 什么是 Hive

Hive 是基于 Hadoop 的**数据仓库工具**，可以将结构化的数据文件映射为一张数据库表，并提供类 SQL 查询功能（HQL）。

**本质**：将 HQL 转化成 MapReduce / Tez / Spark 程序执行。

![Hive架构图](https://cwiki.apache.org/confluence/download/attachments/27362072/system_architecture.png?version=1&modificationDate=1414560669000&api=v2 ':size=700')

### 1.2 Hive 架构 🔥

| 组件 | 功能 |
|:---|:---|
| **用户接口（CLI/JDBC/WebUI）** | 提供 HQL 输入入口 |
| **Driver（驱动器）** | 接收 HQL，管理整个执行生命周期 |
| **解析器（Parser）** | 将 HQL 解析为 AST（抽象语法树） |
| **编译器（Compiler）** | 将 AST 编译成逻辑执行计划 |
| **优化器（Optimizer）** | 对逻辑计划进行优化 |
| **执行器（Executor）** | 将计划转为 MR/Tez/Spark 任务执行 |
| **MetaStore（元数据存储）** | 存储表的结构信息（默认 Derby，生产用 MySQL） |

> 💡 **面试问法**：请描述 Hive 的执行流程？
> **答**：HQL → Parser 解析为 AST → Compiler 编译为逻辑计划 → Optimizer 优化 → Executor 转为 MR/Tez 任务提交到 YARN 执行 → 返回结果

### 1.3 Hive 和数据库的区别 🔥

| 对比项 | Hive | RDBMS（如 MySQL） |
|:---|:---|:---|
| 数据规模 | PB 级 | GB~TB 级 |
| 执行引擎 | MR / Tez / Spark | 自有引擎 |
| 数据存储 | HDFS | 本地磁盘 |
| 执行延迟 | 高（适合批处理） | 低（适合 OLTP） |
| 索引 | 有限支持 | 完善的 B+Tree 等 |
| 可扩展性 | 高（横向扩展） | 有限 |
| 事务 | 有限支持（ACID） | 完整 ACID |
| 数据更新 | 不擅长 | 擅长 |

---

## 第2章 Hive 安装与配置 📝

### 2.1 安装要点

1. **前置条件**：Hadoop 集群已搭建并运行
2. **MetaStore 配置**：生产环境必须使用 MySQL 替代默认的 Derby
3. **关键配置文件**：`hive-site.xml`

```xml
<!-- hive-site.xml 核心配置 -->
<configuration>
    <!-- MetaStore 数据库连接 -->
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://hadoop102:3306/metastore?useSSL=false</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>000000</value>
    </property>
    <!-- Hive 元数据存储版本验证 -->
    <property>
        <name>hive.metastore.schema.verification</name>
        <value>false</value>
    </property>
    <!-- 打印表头 -->
    <property>
        <name>hive.cli.print.header</name>
        <value>true</value>
    </property>
    <!-- 打印当前数据库 -->
    <property>
        <name>hive.cli.print.current.db</name>
        <value>true</value>
    </property>
</configuration>
```

### 2.2 初始化元数据库

```bash
# 初始化 MetaStore
schematool -initSchema -dbType mysql -verbose

# 启动 Hive
hive

# 启动 HiveServer2（JDBC 方式）
hiveserver2 &

# 使用 Beeline 连接
beeline -u jdbc:hive2://hadoop102:10000 -n atguigu
```

---

## 第3章 DDL（数据定义语言） ⭐

### 3.1 数据库操作

```sql
-- 创建数据库
CREATE DATABASE IF NOT EXISTS db_hive;

-- 指定存储位置
CREATE DATABASE db_hive2 LOCATION '/db_hive2';

-- 查看数据库
SHOW DATABASES;
SHOW DATABASES LIKE 'db_hive*';
DESC DATABASE db_hive;

-- 切换数据库
USE db_hive;

-- 删除数据库
DROP DATABASE IF EXISTS db_hive2 CASCADE;  -- CASCADE 强制删除（含表）
```

### 3.2 建表语法 🔥

```sql
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name
    [(col_name data_type [COMMENT col_comment], ...)]
    [COMMENT table_comment]
    [PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)]
    [CLUSTERED BY (col_name, ...) [SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS]
    [ROW FORMAT row_format]
    [STORED AS file_format]
    [LOCATION hdfs_path]
    [TBLPROPERTIES (property_name=property_value, ...)]
```

### 3.3 内部表 vs 外部表 🔥🔥

| 对比 | 内部表（Managed Table） | 外部表（External Table） |
|:---|:---|:---|
| 关键字 | 默认 | `EXTERNAL` |
| 删表时 | **删除元数据 + HDFS 数据** | **仅删除元数据，保留 HDFS 数据** |
| 使用场景 | 临时表、中间表 | **生产环境首选**，数据安全 |

> 💡 **面试必问**：内部表和外部表的区别？删除时分别发生什么？
> **答**：内部表删除时元数据和 HDFS 数据都删，外部表删除只删元数据，HDFS 数据保留。**生产中推荐外部表**，防止误删数据。

```sql
-- 创建外部表（推荐）
CREATE EXTERNAL TABLE IF NOT EXISTS student (
    id INT COMMENT '学生ID',
    name STRING COMMENT '姓名',
    age INT COMMENT '年龄'
)
COMMENT '学生表'
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE
LOCATION '/user/hive/warehouse/student';

-- 内部表与外部表互转
ALTER TABLE student SET TBLPROPERTIES('EXTERNAL'='TRUE');   -- 转为外部表
ALTER TABLE student SET TBLPROPERTIES('EXTERNAL'='FALSE');  -- 转为内部表
```

### 3.4 Hive 数据类型

| 分类 | 类型 | 说明 |
|:---|:---|:---|
| 基本类型 | `TINYINT` | 1字节整数 |
| | `SMALLINT` | 2字节整数 |
| | `INT` | 4字节整数 |
| | `BIGINT` | 8字节整数 |
| | `FLOAT` | 单精度浮点 |
| | `DOUBLE` | 双精度浮点 |
| | `DECIMAL(p,s)` | 高精度数字（金额常用）|
| | `STRING` | 字符串（最常用） |
| | `VARCHAR(n)` | 变长字符串 |
| | `BOOLEAN` | 布尔值 |
| | `TIMESTAMP` | 时间戳 |
| | `DATE` | 日期 |
| 复杂类型 | `ARRAY<T>` | 数组 |
| | `MAP<K,V>` | 键值对 |
| | `STRUCT<a:T, b:T>` | 结构体 |

```sql
-- 复杂类型使用示例
CREATE TABLE complex_test (
    name STRING,
    friends ARRAY<STRING>,           -- 数组
    children MAP<STRING, INT>,       -- Map
    address STRUCT<street:STRING, city:STRING>  -- 结构体
)
ROW FORMAT DELIMITED
    FIELDS TERMINATED BY ','
    COLLECTION ITEMS TERMINATED BY '_'
    MAP KEYS TERMINATED BY ':'
STORED AS TEXTFILE;

-- 查询复杂类型
SELECT name,
       friends[0],                   -- 数组取值
       children['xiaoming'],         -- Map取值
       address.city                  -- Struct取值
FROM complex_test;
```

---

## 第4章 DML（数据操作语言） ⭐

### 4.1 数据装载 Load

```sql
-- 从本地加载数据
LOAD DATA LOCAL INPATH '/opt/module/data/student.txt' INTO TABLE student;

-- 从本地加载（覆盖）
LOAD DATA LOCAL INPATH '/opt/module/data/student.txt' OVERWRITE INTO TABLE student;

-- 从 HDFS 加载（剪切）
LOAD DATA INPATH '/user/data/student.txt' INTO TABLE student;
```

### 4.2 Insert 插入

```sql
-- 插入查询结果
INSERT INTO TABLE student
SELECT id, name FROM student_bak;

-- 覆盖插入
INSERT OVERWRITE TABLE student
SELECT id, name FROM student_bak;

-- 多表插入（一次扫描，多次输出）
FROM source_table
INSERT OVERWRITE TABLE target1 SELECT col1, col2 WHERE condition1
INSERT OVERWRITE TABLE target2 SELECT col1, col3 WHERE condition2;
```

### 4.3 Export / Import

```sql
-- 导出到 HDFS
EXPORT TABLE student TO '/user/hive/export/student';

-- 从 HDFS 导入
IMPORT TABLE student2 FROM '/user/hive/export/student';
```

### 4.4 数据排序 🔥

| 关键字 | 说明 | Reducer数量 |
|:---|:---|:---|
| `ORDER BY` | **全局排序**，数据量大时慎用 | 1个 |
| `SORT BY` | 每个 Reducer 内部排序 | 多个 |
| `DISTRIBUTE BY` | 指定分区字段（类似 MR 的 Partition） | 多个 |
| `CLUSTER BY` | = `DISTRIBUTE BY` + `SORT BY`（同一字段） | 多个 |

```sql
-- 全局排序（慎用，只有1个Reducer）
SELECT * FROM emp ORDER BY sal DESC;

-- 每个Reducer内排序
SET mapreduce.job.reduces = 3;
SELECT * FROM emp SORT BY deptno;

-- 先分区再排序（常用！）
SELECT * FROM emp DISTRIBUTE BY deptno SORT BY sal DESC;

-- 等价于
SELECT * FROM emp CLUSTER BY deptno;  -- 只能升序
```

---

## 第5章 查询（核心重点） 🔥🔥

### 5.1 基本查询

```sql
-- 全表查询
SELECT * FROM emp;

-- 选择特定列
SELECT ename, sal FROM emp;

-- 别名
SELECT ename AS name, sal salary FROM emp;

-- 常用函数
SELECT COUNT(*) cnt FROM emp;               -- 总数
SELECT MAX(sal) max_sal FROM emp;           -- 最大值
SELECT MIN(sal) min_sal FROM emp;           -- 最小值
SELECT SUM(sal) sum_sal FROM emp;           -- 求和
SELECT AVG(sal) avg_sal FROM emp;           -- 平均值

-- LIMIT
SELECT * FROM emp LIMIT 5;
SELECT * FROM emp LIMIT 2, 5;  -- 从第3条开始取5条
```

### 5.2 WHERE 条件

```sql
-- 比较运算
SELECT * FROM emp WHERE sal > 1000;
SELECT * FROM emp WHERE sal BETWEEN 500 AND 1000;
SELECT * FROM emp WHERE comm IS NULL;
SELECT * FROM emp WHERE sal IN (1500, 5000);

-- LIKE 模糊查询
SELECT * FROM emp WHERE ename LIKE 'S%';    -- S开头
SELECT * FROM emp WHERE ename LIKE '_M%';   -- 第二个字符是M
SELECT * FROM emp WHERE ename RLIKE '[A]';  -- 正则：包含A
```

### 5.3 JOIN 关联 🔥

```sql
-- 内连接（只取交集）
SELECT e.ename, d.dname
FROM emp e JOIN dept d ON e.deptno = d.deptno;

-- 左外连接
SELECT e.ename, d.dname
FROM emp e LEFT JOIN dept d ON e.deptno = d.deptno;

-- 右外连接
SELECT e.ename, d.dname
FROM emp e RIGHT JOIN dept d ON e.deptno = d.deptno;

-- 满外连接
SELECT e.ename, d.dname
FROM emp e FULL JOIN dept d ON e.deptno = d.deptno;

-- 笛卡尔积（尽量避免！）
SELECT e.ename, d.dname FROM emp e, dept d;

-- 多表连接
SELECT e.ename, d.dname, l.loc_name
FROM emp e
JOIN dept d ON e.deptno = d.deptno
JOIN location l ON d.loc = l.loc;
```

> 💡 **面试提示**：Hive 中 JOIN 只支持等值连接（`=`），不支持非等值连接。多表 JOIN 时，相同 JOIN key 会合并为一个 MR Job。

### 5.4 分组与聚合 🔥

```sql
-- GROUP BY
SELECT deptno, AVG(sal) avg_sal
FROM emp
GROUP BY deptno;

-- HAVING（对分组后的结果过滤）
SELECT deptno, AVG(sal) avg_sal
FROM emp
GROUP BY deptno
HAVING avg_sal > 2000;
```

### 5.5 窗口函数 🔥🔥🔥

> 窗口函数是 **Hive 面试最高频考点之一**！

#### 5.5.1 语法结构

```sql
函数名(参数) OVER (
    [PARTITION BY 分区列]
    [ORDER BY 排序列]
    [ROWS BETWEEN 窗口起始 AND 窗口结束]
)
```

#### 5.5.2 排名函数（面试必会）

```sql
-- 准备数据
CREATE TABLE score (
    name STRING, subject STRING, score INT
);

-- 三大排名函数对比 🔥🔥🔥
SELECT
    name, subject, score,
    ROW_NUMBER() OVER(PARTITION BY subject ORDER BY score DESC) AS rn,    -- 1,2,3,4
    RANK()       OVER(PARTITION BY subject ORDER BY score DESC) AS rk,    -- 1,2,2,4（跳号）
    DENSE_RANK() OVER(PARTITION BY subject ORDER BY score DESC) AS dr     -- 1,2,2,3（不跳号）
FROM score;
```

| 函数 | 相同值排名 | 后续排名 | 示例 |
|:---|:---|:---|:---|
| `ROW_NUMBER()` | 不同 | 连续 | 1,2,3,4,5 |
| `RANK()` | 相同 | **跳号** | 1,2,2,4,5 |
| `DENSE_RANK()` | 相同 | **不跳号** | 1,2,2,3,4 |

> 💡 **面试经典题**：每个科目取前3名？
```sql
SELECT * FROM (
    SELECT name, subject, score,
           ROW_NUMBER() OVER(PARTITION BY subject ORDER BY score DESC) AS rn
    FROM score
) t
WHERE rn <= 3;
```

#### 5.5.3 聚合窗口函数

```sql
SELECT
    name, orderdate, cost,
    SUM(cost) OVER() AS total,                                                     -- 所有行总和
    SUM(cost) OVER(PARTITION BY name) AS name_total,                              -- 按name分组总和
    SUM(cost) OVER(PARTITION BY name ORDER BY orderdate) AS name_cumsum,          -- 累计求和
    SUM(cost) OVER(PARTITION BY name ORDER BY orderdate ROWS BETWEEN 1 PRECEDING AND CURRENT ROW) AS last2  -- 当前行+前1行
FROM business;
```

#### 5.5.4 LAG / LEAD 偏移函数 🔥

```sql
-- LAG：向前取值（取前一行）
-- LEAD：向后取值（取后一行）
SELECT
    name, orderdate, cost,
    LAG(orderdate, 1, '1970-01-01')  OVER(PARTITION BY name ORDER BY orderdate) AS prev_date,
    LEAD(orderdate, 1, '9999-12-31') OVER(PARTITION BY name ORDER BY orderdate) AS next_date
FROM business;
```

#### 5.5.5 NTILE 分片函数

```sql
-- 将数据分成N片
SELECT
    name, orderdate, cost,
    NTILE(5) OVER(ORDER BY orderdate) AS groupid
FROM business;

-- 取前20%的数据
SELECT * FROM (
    SELECT *, NTILE(5) OVER(ORDER BY orderdate) AS groupid
    FROM business
) t
WHERE groupid = 1;
```

### 5.6 常用内置函数 🔥

#### 字符串函数

```sql
SELECT LENGTH('hello');                        -- 5
SELECT CONCAT('hello', '-', 'world');          -- hello-world
SELECT CONCAT_WS('-', 'hello', 'world');       -- hello-world
SELECT SUBSTR('hello', 1, 3);                  -- hel
SELECT UPPER('hello');                         -- HELLO
SELECT LOWER('HELLO');                         -- hello
SELECT TRIM('  hello  ');                      -- hello
SELECT REGEXP_REPLACE('hello123', '[0-9]', ''); -- hello
SELECT SPLIT('hello,world', ',');               -- ["hello","world"]
SELECT GET_JSON_OBJECT('{"name":"zhangsan"}', '$.name'); -- zhangsan
```

#### 日期函数

```sql
SELECT CURRENT_DATE();                         -- 当前日期
SELECT CURRENT_TIMESTAMP();                    -- 当前时间戳
SELECT DATEDIFF('2023-12-31', '2023-01-01');   -- 364
SELECT DATE_ADD('2023-01-01', 10);             -- 2023-01-11
SELECT DATE_SUB('2023-01-01', 10);             -- 2022-12-22
SELECT DATE_FORMAT('2023-01-01', 'yyyy/MM/dd'); -- 2023/01/01
SELECT YEAR('2023-06-15');                     -- 2023
SELECT MONTH('2023-06-15');                    -- 6
SELECT DAY('2023-06-15');                      -- 15
```

#### 集合函数

```sql
SELECT SIZE(ARRAY(1,2,3));                     -- 3
SELECT SIZE(MAP('a',1,'b',2));                 -- 2
SELECT ARRAY_CONTAINS(ARRAY(1,2,3), 2);        -- true
SELECT SORT_ARRAY(ARRAY(3,1,2));               -- [1,2,3]
SELECT COLLECT_SET(col) FROM table;            -- 去重收集为数组
SELECT COLLECT_LIST(col) FROM table;           -- 不去重收集为数组
```

#### 条件函数

```sql
-- CASE WHEN
SELECT name,
    CASE
        WHEN sal > 5000 THEN '高薪'
        WHEN sal > 3000 THEN '中薪'
        ELSE '低薪'
    END AS salary_level
FROM emp;

-- IF
SELECT name, IF(sal > 3000, '高薪', '低薪') FROM emp;

-- COALESCE（返回第一个非null值）
SELECT COALESCE(comm, 0) FROM emp;

-- NVL（同 COALESCE 简化版）
SELECT NVL(comm, 0) FROM emp;
```

#### 行转列 / 列转行 🔥

```sql
-- 行转列：CONCAT_WS + COLLECT_SET
SELECT dept,
       CONCAT_WS('|', COLLECT_SET(name)) AS names
FROM emp
GROUP BY dept;

-- 列转行：LATERAL VIEW + EXPLODE
SELECT name, hobby
FROM person
LATERAL VIEW EXPLODE(hobbies) tmp AS hobby;

-- EXPLODE + POSEXPLODE
SELECT name, pos, hobby
FROM person
LATERAL VIEW POSEXPLODE(hobbies) tmp AS pos, hobby;
```

---

## 第6章 分区表与分桶表 🔥

### 6.1 分区表 🔥🔥

分区表通过**目录划分**数据，查询时只扫描指定分区，大幅提升效率。

```sql
-- 创建分区表
CREATE TABLE dept_partition (
    deptno INT, dname STRING, loc STRING
)
PARTITIONED BY (day STRING)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t';

-- 加载数据到指定分区
LOAD DATA LOCAL INPATH '/opt/module/data/dept_20230101.log'
INTO TABLE dept_partition PARTITION(day='20230101');

-- 查询指定分区
SELECT * FROM dept_partition WHERE day = '20230101';

-- 增加分区
ALTER TABLE dept_partition ADD PARTITION(day='20230102');

-- 删除分区
ALTER TABLE dept_partition DROP PARTITION(day='20230102');

-- 查看分区
SHOW PARTITIONS dept_partition;
```

#### 二级分区

```sql
CREATE TABLE dept_partition2 (
    deptno INT, dname STRING, loc STRING
)
PARTITIONED BY (day STRING, hour STRING)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t';

-- 加载数据
LOAD DATA LOCAL INPATH '/data/dept.log'
INTO TABLE dept_partition2 PARTITION(day='20230101', hour='12');
```

#### 动态分区 🔥

```sql
-- 开启动态分区
SET hive.exec.dynamic.partition = true;
SET hive.exec.dynamic.partition.mode = nonstrict;  -- 允许全部动态分区
SET hive.exec.max.dynamic.partitions = 1000;       -- 最大动态分区数
SET hive.exec.max.dynamic.partitions.pernode = 100; -- 每个节点最大分区数

-- 动态分区插入（分区字段放最后）
INSERT OVERWRITE TABLE dept_partition PARTITION(day)
SELECT deptno, dname, loc, day FROM dept_tmp;
```

### 6.2 分桶表 ⭐

分桶是对**文件**进行划分（Hash 分桶），适用于抽样查询和优化 JOIN。

```sql
-- 创建分桶表
CREATE TABLE stu_bucket (
    id INT, name STRING
)
CLUSTERED BY(id) INTO 4 BUCKETS
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t';

-- 抽样查询
-- TABLESAMPLE(BUCKET x OUT OF y ON id)
-- 含义：分y桶，取第x桶
SELECT * FROM stu_bucket TABLESAMPLE(BUCKET 1 OUT OF 4 ON id);
```

> 💡 **分区 vs 分桶**：分区是按目录划分，分桶是按文件划分。分区适合按时间等维度过滤，分桶适合 JOIN 优化和抽样。

---

## 第7章 文件格式与压缩 🔥

### 7.1 文件格式对比

| 格式 | 类型 | 特点 | 适用场景 |
|:---|:---|:---|:---|
| **TextFile** | 行式 | 默认格式，可读性好，压缩比低 | 小数据、调试 |
| **SequenceFile** | 行式 | 二进制 KV，支持压缩 | 少用 |
| **ORC** | **列式** 🔥 | Hive 专用，压缩比高，查询快 | **Hive 首选** |
| **Parquet** | **列式** 🔥 | 跨平台通用（Spark/Flink/Impala） | **跨引擎场景** |

### 7.2 ORC vs Parquet 🔥

| 对比项 | ORC | Parquet |
|:---|:---|:---|
| 开发方 | Hortonworks（Hive） | Twitter + Cloudera |
| 生态 | Hive 最优 | Spark / Flink / Impala |
| 压缩比 | 更高 | 较高 |
| 嵌套结构 | 一般 | 更好 |
| 索引 | 内置三级索引 | 行组+列统计 |
| **建议** | **纯 Hive 环境用 ORC** | **跨引擎用 Parquet** |

```sql
-- 创建 ORC 表
CREATE TABLE log_orc (
    id INT, name STRING, ts BIGINT
)
STORED AS ORC
TBLPROPERTIES("orc.compress"="SNAPPY");

-- 创建 Parquet 表
CREATE TABLE log_parquet (
    id INT, name STRING, ts BIGINT
)
STORED AS PARQUET;
```

### 7.3 压缩方式 ⭐

| 压缩格式 | 可切分 | 压缩比 | 速度 | 推荐度 |
|:---|:---:|:---|:---|:---:|
| **Snappy** | 否 | 中 | **快** | 🔥 常用 |
| **LZO** | 是（需索引） | 中 | 快 | ⭐ |
| **GZIP** | 否 | **高** | 慢 | 最终存储 |
| **ZLIB** | 否 | 高 | 慢 | ORC默认 |
| **Zstd** | 否 | 高 | 较快 | 🔥 新选择 |

```sql
-- 设置 Map 输出压缩
SET mapreduce.map.output.compress = true;
SET mapreduce.map.output.compress.codec = org.apache.hadoop.io.compress.SnappyCodec;

-- 设置最终输出压缩
SET mapreduce.output.fileoutputformat.compress = true;
SET mapreduce.output.fileoutputformat.compress.codec = org.apache.hadoop.io.compress.SnappyCodec;

-- Hive 中间结果压缩
SET hive.exec.compress.intermediate = true;
SET hive.exec.compress.output = true;
```

---

## 第8章 自定义函数 UDF 🔥

### 8.1 函数分类

| 类型 | 输入行数 | 输出行数 | 示例 |
|:---|:---|:---|:---|
| **UDF** | 1 | 1 | `upper()`, `substr()` |
| **UDAF** | 多 | 1 | `count()`, `sum()` |
| **UDTF** | 1 | 多 | `explode()`, `lateral view` |

### 8.2 自定义 UDF 示例 🔥

```java
import org.apache.hadoop.hive.ql.exec.UDFArgumentException;
import org.apache.hadoop.hive.ql.metadata.HiveException;
import org.apache.hadoop.hive.ql.udf.generic.GenericUDF;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.primitive.PrimitiveObjectInspectorFactory;

/**
 * 自定义 UDF：计算字符串长度
 * 使用方式：SELECT my_len('hello') => 5
 */
public class MyUDF extends GenericUDF {

    @Override
    public ObjectInspector initialize(ObjectInspector[] arguments) throws UDFArgumentException {
        if (arguments.length != 1) {
            throw new UDFArgumentException("只接受一个参数");
        }
        return PrimitiveObjectInspectorFactory.javaIntObjectInspector;
    }

    @Override
    public Object evaluate(DeferredObject[] arguments) throws HiveException {
        if (arguments[0].get() == null) {
            return 0;
        }
        return arguments[0].get().toString().length();
    }

    @Override
    public String getDisplayString(String[] children) {
        return "my_len(string)";
    }
}
```

```sql
-- 注册 UDF
-- 1. 将 jar 包上传到 HDFS
-- 2. 创建函数
ADD JAR hdfs:///user/hive/jars/my_udf.jar;
CREATE TEMPORARY FUNCTION my_len AS 'com.atguigu.hive.MyUDF';

-- 使用
SELECT my_len('hello world');  -- 11

-- 永久函数
CREATE FUNCTION my_len AS 'com.atguigu.hive.MyUDF' USING JAR 'hdfs:///user/hive/jars/my_udf.jar';
```

---

## 第9章 企业级调优 🔥🔥🔥

> 这是 **Hive 面试的重灾区**！几乎每次面试都会问到！

### 9.1 Explain 执行计划 🔥

```sql
-- 查看执行计划
EXPLAIN SELECT deptno, AVG(sal) FROM emp GROUP BY deptno;

-- 查看详细执行计划
EXPLAIN EXTENDED SELECT deptno, AVG(sal) FROM emp GROUP BY deptno;

-- 关注点：
-- 1. Stage数量（越少越好）
-- 2. Map/Reduce 数量
-- 3. 是否有数据倾斜风险
```

### 9.2 Map 数量优化 ⭐

```sql
-- 小文件合并（增大Map处理量）
SET hive.input.format = org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;

-- 调整 Map 数量
SET mapreduce.input.fileinputformat.split.maxsize = 256000000;  -- 256MB
```

### 9.3 Reduce 数量优化 ⭐

```sql
-- 设置 Reduce 数量
SET mapreduce.job.reduces = 15;

-- 或者设置每个 Reduce 处理的数据量
SET hive.exec.reducers.bytes.per.reducer = 256000000;  -- 256MB
```

### 9.4 MapJoin（小表广播） 🔥🔥

```sql
-- 自动 MapJoin（默认开启）
SET hive.auto.convert.join = true;
SET hive.mapjoin.smalltable.filesize = 25000000;  -- 小表阈值25MB

-- 手动指定 MapJoin
SELECT /*+ MAPJOIN(small_table) */
    a.key, a.value, b.value
FROM big_table a
JOIN small_table b ON a.key = b.key;
```

> 💡 **面试问法**：Hive 中大表 JOIN 小表如何优化？
> **答**：使用 MapJoin，将小表加载到内存中广播到每个 Map 任务，避免 Reduce 阶段的 Shuffle，大幅提升效率。

### 9.5 数据倾斜 🔥🔥🔥

> **数据倾斜**是大数据面试的终极考题！

#### 表现
- 大部分 Reduce 已完成，少数 Reduce 卡在 99%
- 某些 Reduce 处理数据量远超其他

#### 原因
1. key 分布不均匀（如 null 值、热点 key）
2. 业务数据本身不均匀

#### 解决方案

**方案1：开启数据倾斜自动优化**
```sql
-- 开启 Skew Join
SET hive.optimize.skewjoin = true;
SET hive.skewjoin.key = 100000;  -- 超过10万条认为是倾斜key

-- 开启 Group By 数据倾斜优化
SET hive.groupby.skewindata = true;
-- 原理：两个MR Job，第一个随机分发，第二个按key聚合
```

**方案2：null 值处理**
```sql
-- 方法一：过滤null
SELECT * FROM log a JOIN user b ON a.user_id = b.user_id
WHERE a.user_id IS NOT NULL;

-- 方法二：给null赋随机值
SELECT * FROM log a
LEFT JOIN user b
ON COALESCE(a.user_id, CONCAT('rand_', RAND())) = b.user_id;
```

**方案3：热点 key 打散**
```sql
-- 给倾斜key加随机前缀
SELECT a.key, SUM(a.val)
FROM (
    SELECT CONCAT(key, '_', CAST(FLOOR(RAND() * 10) AS STRING)) AS key, val
    FROM big_table
) a
GROUP BY a.key;
```

**方案4：Map-Side 预聚合**
```sql
SET hive.map.aggr = true;  -- 默认开启
SET hive.groupby.mapaggr.checkinterval = 100000;
```

### 9.6 小文件问题 🔥

```sql
-- Map 端合并小文件
SET hive.input.format = org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;

-- Reduce 端合并小文件
SET hive.merge.mapfiles = true;        -- Map-only 任务合并
SET hive.merge.mapredfiles = true;     -- MapReduce 任务合并
SET hive.merge.size.per.task = 256000000;       -- 合并后文件大小
SET hive.merge.smallfiles.avgsize = 16000000;   -- 触发合并的平均文件大小
```

### 9.7 其他优化 ⭐

```sql
-- 开启矢量化查询（ORC格式）
SET hive.vectorized.execution.enabled = true;
SET hive.vectorized.execution.reduce.enabled = true;

-- 开启 CBO 基于成本的优化
SET hive.cbo.enable = true;
SET hive.compute.query.using.stats = true;
SET hive.stats.fetch.column.stats = true;

-- 开启谓词下推
SET hive.optimize.ppd = true;

-- 开启 Fetch 抓取（简单查询不走 MR）
SET hive.fetch.task.conversion = more;

-- 开启并行执行
SET hive.exec.parallel = true;
SET hive.exec.parallel.thread.number = 16;

-- 使用 Tez 引擎
SET hive.execution.engine = tez;
```

---

## 第10章 Hive 高频面试题 🔥🔥🔥

### Q1：Hive 内部表和外部表的区别？
> 内部表（managed table）：删除表时，**元数据和 HDFS 数据都删除**。
> 外部表（external table）：删除表时，**只删除元数据，保留 HDFS 数据**。
> **生产中统一使用外部表**，防止误删数据。

### Q2：Hive 中 Order By、Sort By、Distribute By、Cluster By 的区别？
> - `ORDER BY`：全局排序，只有 1 个 Reducer
> - `SORT BY`：每个 Reducer 内部排序
> - `DISTRIBUTE BY`：指定数据分区规则（类似 MR 的 Partitioner）
> - `CLUSTER BY`：= `DISTRIBUTE BY` + `SORT BY`（字段相同时使用）

### Q3：Hive 的窗口函数 ROW_NUMBER、RANK、DENSE_RANK 区别？
> - `ROW_NUMBER()`：连续排名 1,2,3,4,5
> - `RANK()`：相同值相同排名，**跳号** 1,2,2,4,5
> - `DENSE_RANK()`：相同值相同排名，**不跳号** 1,2,2,3,4

### Q4：Hive 数据倾斜怎么解决？
> 1. 开启 `hive.optimize.skewjoin = true`
> 2. 开启 `hive.groupby.skewindata = true`
> 3. MapJoin 避免 Reduce 端 Join
> 4. null 值赋随机值 / 过滤
> 5. 热点 key 加随机前缀打散
> 6. Map 端预聚合

### Q5：ORC 和 Parquet 怎么选？
> 纯 Hive 环境用 **ORC**（压缩比更高，内置索引），跨引擎（Spark/Flink）用 **Parquet**。

### Q6：Hive SQL 执行流程？
> HQL → Parser(AST) → Compiler(逻辑计划) → Optimizer(优化) → Executor(MR/Tez任务) → YARN执行 → 返回结果

### Q7：Hive 如何处理小文件？
> 1. `CombineHiveInputFormat` 合并 Map 端输入
> 2. `hive.merge.mapfiles` / `hive.merge.mapredfiles` 合并输出
> 3. 使用 ORC/Parquet 列式存储，Snappy 压缩

### Q8：Hive 的 UDF、UDAF、UDTF 区别？
> - UDF：一进一出（如 `upper()`）
> - UDAF：多进一出（如 `count()`、`sum()`）
> - UDTF：一进多出（如 `explode()`）
