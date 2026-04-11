# Python

> 📌 **重点**：大数据开发中 Python 快速入门

---

## 第1章 基础语法 ⭐

### 1.1 变量与类型

```python
# 变量不需要声明类型
name = "张三"
age = 25
salary = 8888.88
is_active = True

# 类型转换
str(100)      # "100"
int("100")    # 100
float("3.14") # 3.14

# 格式化字符串
print(f"姓名: {name}, 年龄: {age}")
```

### 1.2 数据结构

```python
# List（类似 Java ArrayList）
fruits = ["apple", "banana", "cherry"]
fruits.append("date")
fruits[0]        # "apple"
fruits[1:3]      # 切片

# Dict（类似 Java HashMap）
user = {"name": "张三", "age": 25}
user["name"]     # "张三"
user.get("phone", "无")

# Set
s = {1, 2, 3, 3}  # {1, 2, 3}

# Tuple（不可变）
t = (1, 2, 3)

# 列表推导式 🔥
squares = [x**2 for x in range(10)]
evens = [x for x in range(20) if x % 2 == 0]
```

### 1.3 函数

```python
def add(a, b=0):
    """两数相加"""
    return a + b

# Lambda
double = lambda x: x * 2

# map / filter
list(map(lambda x: x * 2, [1, 2, 3]))     # [2, 4, 6]
list(filter(lambda x: x > 2, [1, 2, 3]))  # [3]
```

---

## 第2章 在大数据中的应用 ⭐

### 2.1 PySpark 示例

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("WordCount") \
    .master("local[*]") \
    .getOrCreate()

# 读取文件
df = spark.read.text("hdfs://hadoop102:8020/input/words.txt")

# WordCount
from pyspark.sql.functions import explode, split, col
result = df.select(explode(split(col("value"), " ")).alias("word")) \
           .groupBy("word") \
           .count() \
           .orderBy("count", ascending=False)

result.show()
spark.stop()
```

### 2.2 常用数据分析库

| 库 | 用途 |
|:---|:---|
| **pandas** | 数据处理与分析 |
| **numpy** | 数值计算 |
| **matplotlib** | 数据可视化 |
| **PySpark** | Spark Python API |

---

## 第3章 文件与异常 ⭐

### 3.1 文件操作

```python
# 读取文件
with open("data.txt", "r", encoding="utf-8") as f:
    lines = f.readlines()    # 返回列表
    # 或逐行读取
    for line in f:
        print(line.strip())

# 写入文件
with open("output.txt", "w", encoding="utf-8") as f:
    f.write("Hello\n")
    f.writelines(["Line1\n", "Line2\n"])

# 读取 JSON
import json
with open("config.json", "r") as f:
    config = json.load(f)

# 写入 JSON
with open("result.json", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)

# 读取 CSV
import csv
with open("data.csv", "r") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row["name"], row["age"])
```

### 3.2 异常处理

```python
try:
    result = 10 / 0
except ZeroDivisionError as e:
    print(f"除零错误: {e}")
except Exception as e:
    print(f"其他错误: {e}")
finally:
    print("清理资源")

# 自定义异常
class DataException(Exception):
    def __init__(self, msg):
        super().__init__(msg)

raise DataException("数据格式错误")
```

---

## 第4章 大数据脚本实战 🔥

### 4.1 DataX 配置生成器

```python
#!/usr/bin/env python
"""批量生成 DataX 配置文件"""
import json, sys, os

def gen_config(db, table, dt):
    return {
        "job": {
            "setting": {"speed": {"channel": 3}},
            "content": [{
                "reader": {
                    "name": "mysqlreader",
                    "parameter": {
                        "username": "root",
                        "password": "000000",
                        "column": ["*"],
                        "connection": [{
                            "table": [table],
                            "jdbcUrl": [f"jdbc:mysql://hadoop102:3306/{db}"]
                        }]
                    }
                },
                "writer": {
                    "name": "hdfswriter",
                    "parameter": {
                        "defaultFS": "hdfs://hadoop102:8020",
                        "fileType": "orc",
                        "path": f"/origin_data/{db}/db/{table}/{dt}",
                        "fileName": table,
                        "writeMode": "append"
                    }
                }
            }]
        }
    }

if __name__ == "__main__":
    tables = ["user_info", "order_info", "sku_info"]
    for t in tables:
        config = gen_config("gmall", t, "${dt}")
        with open(f"{t}.json", "w") as f:
            json.dump(config, f, indent=4)
        print(f"✅ Generated {t}.json")
```

### 4.2 集群启停脚本

```python
#!/usr/bin/env python
"""集群组件批量启停"""
import subprocess, sys

HOSTS = ["hadoop102", "hadoop103", "hadoop104"]

def ssh_cmd(host, cmd):
    """远程执行命令"""
    result = subprocess.run(
        ["ssh", host, cmd],
        capture_output=True, text=True
    )
    print(f"[{host}] {result.stdout.strip()}")
    return result.returncode

def start_cluster():
    print("========== 启动 ZooKeeper ==========")
    for host in HOSTS:
        ssh_cmd(host, "/opt/module/zookeeper/bin/zkServer.sh start")
    
    print("========== 启动 HDFS ==========")
    ssh_cmd("hadoop102", "start-dfs.sh")
    
    print("========== 启动 YARN ==========")
    ssh_cmd("hadoop103", "start-yarn.sh")

if __name__ == "__main__":
    action = sys.argv[1] if len(sys.argv) > 1 else "start"
    if action == "start":
        start_cluster()
```

### 4.3 日志分析脚本

```python
"""分析 Hadoop 日志中的错误信息"""
import re
from collections import Counter

error_pattern = re.compile(r"(\w+Exception|ERROR\s+\w+)")

with open("/opt/module/hadoop/logs/hadoop.log", "r") as f:
    errors = []
    for line in f:
        match = error_pattern.search(line)
        if match:
            errors.append(match.group(1))

# 统计 Top 10 错误
for error, count in Counter(errors).most_common(10):
    print(f"{error}: {count} 次")
```

---

## 第5章 面试要点 🔥

### Q1：Python 中 list 和 tuple 的区别？
> list 可变，tuple 不可变。tuple 作为 dict 的 key、性能更好。

### Q2：Python 中的 GIL 是什么？
> 全局解释器锁（Global Interpreter Lock），同一时刻只有一个线程执行 Python 字节码。CPU 密集型任务用多进程，IO 密集型用多线程或协程。

### Q3：Python 在大数据中的应用场景？
> 1. **PySpark**：Spark 数据处理
> 2. **脚本工具**：DataX 配置生成、集群管理
> 3. **数据分析**：pandas + numpy + matplotlib
> 4. **ETL 脚本**：数据清洗、格式转换
> 5. **调度脚本**：DolphinScheduler/Airflow 中的 Python 算子
