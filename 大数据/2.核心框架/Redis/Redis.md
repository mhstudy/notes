# Redis

> 🔗 **官方网站**：https://redis.io/
> 📖 **官方文档**：https://redis.io/docs/
> 📌 **学习版本**：Redis 7.x

---

## 第1章 Redis 概述 ⭐

### 1.1 什么是 Redis

Redis（Remote Dictionary Server）是一个开源的**内存数据库**，支持多种数据结构，常用作缓存、消息队列、分布式锁等。

### 1.2 特点

- **纯内存操作**：读写速度极快（10万+ QPS）
- **单线程模型**（6.0 前）：避免线程切换和竞争
- **IO 多路复用**：epoll 事件驱动
- **丰富的数据结构**：String、List、Set、Hash、Sorted Set 等

---

## 第2章 五大数据类型 🔥

### 2.1 String（字符串）

```bash
SET name "zhangsan"         # 设置值
GET name                    # 获取值
SETNX lock "1"              # 不存在才设置（分布式锁）
SET key value EX 30         # 设置值并设过期时间30秒
INCR counter                # 自增1
INCRBY counter 10           # 自增10
MSET k1 v1 k2 v2           # 批量设置
MGET k1 k2                 # 批量获取
```

### 2.2 List（列表）

```bash
LPUSH list a b c            # 左侧插入
RPUSH list d e              # 右侧插入
LRANGE list 0 -1            # 获取所有元素
LPOP list                   # 左侧弹出
RPOP list                   # 右侧弹出
LLEN list                   # 长度
LINDEX list 0               # 按索引获取
```

### 2.3 Set（集合）

```bash
SADD set1 a b c             # 添加元素
SMEMBERS set1               # 获取所有元素
SISMEMBER set1 a            # 判断是否存在
SCARD set1                  # 元素个数
SINTER set1 set2            # 交集
SUNION set1 set2            # 并集
SDIFF set1 set2             # 差集
```

### 2.4 Hash（哈希）

```bash
HSET user:1001 name "zhangsan" age "25"   # 设置字段
HGET user:1001 name          # 获取单个字段
HGETALL user:1001            # 获取所有字段
HDEL user:1001 age           # 删除字段
HINCRBY user:1001 age 1      # 字段自增
HKEYS user:1001              # 获取所有 key
HVALS user:1001              # 获取所有 value
```

### 2.5 Sorted Set（有序集合）🔥

```bash
ZADD rank 100 "user1" 200 "user2" 150 "user3"   # 添加（带分数）
ZRANGE rank 0 -1 WITHSCORES                       # 升序获取
ZREVRANGE rank 0 -1 WITHSCORES                    # 降序获取
ZRANGEBYSCORE rank 100 200                        # 按分数范围查询
ZSCORE rank "user1"                               # 获取分数
ZRANK rank "user1"                                # 获取排名
ZCARD rank                                        # 元素个数
```

---

## 第3章 缓存三大问题 🔥🔥🔥

> **面试必问！几乎100%会被问到！**

### 3.1 缓存穿透 🔥

**定义**：查询一个**数据库中也不存在**的数据，每次都穿透缓存直接查数据库。

**解决方案**：

| 方案 | 说明 |
|:---|:---|
| **缓存空对象** | 查不到的 key 也缓存（value=null），设短过期时间 |
| **布隆过滤器** 🔥 | 在缓存前加一层布隆过滤器，不存在的 key 直接拦截 |

```java
// 缓存空对象方案
public String getData(String key) {
    String value = redis.get(key);
    if (value != null) {
        return "null".equals(value) ? null : value;
    }
    value = db.query(key);
    if (value == null) {
        redis.setex(key, 300, "null");  // 缓存空值5分钟
    } else {
        redis.set(key, value);
    }
    return value;
}
```

### 3.2 缓存击穿 🔥

**定义**：某个**热点 key 过期**的瞬间，大量请求同时打到数据库。

**解决方案**：

| 方案 | 说明 |
|:---|:---|
| **互斥锁** | 只允许一个线程查数据库并回填缓存 |
| **逻辑过期** | 不设 TTL，在 value 中存过期时间，由后台线程刷新 |
| **热点 key 永不过期** | 手动管理热点数据 |

```java
// 互斥锁方案
public String getDataWithLock(String key) {
    String value = redis.get(key);
    if (value != null) return value;
    
    String lockKey = "lock:" + key;
    if (redis.setnx(lockKey, "1")) {  // 获取锁
        redis.expire(lockKey, 10);     // 设锁超时
        try {
            value = db.query(key);
            redis.set(key, value);
        } finally {
            redis.del(lockKey);        // 释放锁
        }
    } else {
        Thread.sleep(100);
        return getDataWithLock(key);   // 重试
    }
    return value;
}
```

### 3.3 缓存雪崩 🔥

**定义**：**大量 key 同时过期**或 Redis 宕机，导致请求全部打到数据库。

**解决方案**：

| 方案 | 说明 |
|:---|:---|
| **过期时间加随机值** | 避免同时过期：`TTL = base + random(0, 300)` |
| **Redis 集群 / 哨兵** | 保证 Redis 高可用 |
| **限流降级** | Hystrix / Sentinel 熔断降级 |
| **多级缓存** | 本地缓存(Caffeine) + Redis + DB |

---

## 第4章 持久化 🔥

### 4.1 RDB（快照）

```bash
# 配置文件 redis.conf
save 3600 1         # 3600秒内有1次修改则持久化
save 300 100        # 300秒内有100次修改则持久化
save 60 10000       # 60秒内有10000次修改则持久化
dbfilename dump.rdb
```

| 优点 | 缺点 |
|:---|:---|
| 恢复速度快 | 可能丢失最后一次快照后的数据 |
| 适合全量备份 | fork 子进程消耗内存 |

### 4.2 AOF（追加日志）

```bash
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec    # 每秒同步（推荐）
# always   每次写操作同步（最安全，最慢）
# no       由操作系统决定
```

| 优点 | 缺点 |
|:---|:---|
| 数据安全性高 | 文件大，恢复慢 |
| 支持 AOF 重写压缩 | |

> 💡 **面试答法**：生产环境建议 **RDB + AOF 混合持久化**（Redis 4.0+），兼顾速度和安全。

---

## 第5章 高可用 ⭐

### 5.1 主从复制

```
Master ──写──▶ Slave1（只读）
         └──▶ Slave2（只读）
```

### 5.2 哨兵模式（Sentinel）🔥

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Sentinel1│  │ Sentinel2│  │ Sentinel3│
└─────┬────┘  └─────┬────┘  └─────┬────┘
      │             │             │
      ▼             ▼             ▼
   Master ◀──── Slave1 ◀──── Slave2
```

哨兵功能：**监控、通知、自动故障转移**

### 5.3 Redis Cluster 集群

- 数据分片：16384 个 slot 分布在不同节点
- 去中心化：任一节点都可以接收请求
- 自动故障转移

---

## 第6章 在大数据项目中的应用 🔥

| 场景 | 说明 |
|:---|:---|
| **维度数据缓存** | 将 MySQL/HBase 维度表缓存到 Redis，减少查询延迟 |
| **实时去重** | 使用 Set/HyperLogLog 进行 UV 去重 |
| **实时排行榜** | 使用 Sorted Set 实现 |
| **分布式锁** | 使用 SETNX + 过期时间 |
| **旁路缓存** | Flink 读取 Redis 作为旁路缓存关联维度信息 |

---

## 第7章 高频面试题 🔥🔥🔥

### Q1：缓存穿透、击穿、雪崩的区别？
> - **穿透**：查不存在的数据 → 布隆过滤器/缓存空值
> - **击穿**：热点key过期 → 互斥锁/逻辑过期
> - **雪崩**：大量key同时过期 → 随机TTL/集群/降级

### Q2：Redis 为什么这么快？
> 纯内存操作 + 单线程（无锁竞争）+ IO多路复用 + 高效数据结构

### Q3：RDB 和 AOF 的区别？
> RDB：定期快照，恢复快，可能丢数据。AOF：每次操作记录日志，数据安全，文件大。推荐混合使用。
