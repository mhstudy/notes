# MySQL

> 📌 **重点**：大数据面试中的 MySQL 核心知识点

---

## 第1章 SQL 基础 ⭐

### 1.1 DDL

```sql
-- 建库
CREATE DATABASE IF NOT EXISTS gmall DEFAULT CHARSET utf8mb4;
USE gmall;

-- 建表
CREATE TABLE user_info (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL COMMENT '姓名',
    age INT DEFAULT 0,
    email VARCHAR(100) UNIQUE,
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_name(name)    -- 普通索引
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';

-- 修改表
ALTER TABLE user_info ADD COLUMN phone VARCHAR(20) AFTER email;
ALTER TABLE user_info MODIFY COLUMN name VARCHAR(100);
ALTER TABLE user_info DROP COLUMN phone;
```

### 1.2 DML

```sql
INSERT INTO user_info(name, age, email) VALUES('张三', 25, 'zs@qq.com');
UPDATE user_info SET age = 26 WHERE name = '张三';
DELETE FROM user_info WHERE id = 1;
```

### 1.3 查询 🔥

```sql
-- 连接查询
SELECT u.name, o.order_no
FROM user_info u
INNER JOIN order_info o ON u.id = o.user_id;

-- 子查询
SELECT * FROM user_info WHERE id IN (SELECT user_id FROM order_info);

-- 聚合
SELECT age, COUNT(*) cnt FROM user_info GROUP BY age HAVING cnt > 5;

-- 窗口函数（MySQL 8.0+）
SELECT name, age,
    ROW_NUMBER() OVER(ORDER BY age DESC) AS rn,
    RANK() OVER(ORDER BY age DESC) AS rk
FROM user_info;
```

---

## 第2章 索引 🔥🔥🔥

### 2.1 索引类型

| 类型 | 说明 |
|:---|:---|
| **B+Tree 索引** | InnoDB 默认，最常用 |
| **Hash 索引** | Memory 引擎，等值查询快 |
| **全文索引** | FULLTEXT |

### 2.2 B+Tree 索引原理 🔥

- 所有数据存储在**叶子节点**（非叶子节点只存索引）
- 叶子节点通过**双向链表**连接（范围查询高效）
- 树高一般 3~4 层，千万级数据只需 3~4 次 IO

### 2.3 索引优化 🔥

```sql
-- 查看执行计划
EXPLAIN SELECT * FROM user_info WHERE name = '张三';

-- 关注 type 列：
-- system > const > eq_ref > ref > range > index > ALL
-- ALL 全表扫描最差！
```

**索引失效场景**（面试高频！）：
1. `LIKE '%xxx'` 左模糊
2. 对索引列使用函数或运算
3. 类型隐式转换
4. `OR` 条件中有未索引列
5. `NOT IN`、`NOT EXISTS`
6. 违反最左前缀原则

---

## 第3章 事务 🔥

### 3.1 ACID 特性

| 特性 | 说明 |
|:---|:---|
| **A（原子性）** | 事务全部成功或全部回滚 |
| **C（一致性）** | 事务前后数据一致 |
| **I（隔离性）** | 事务间互不影响 |
| **D（持久性）** | 事务提交后数据永久保存 |

### 3.2 隔离级别 🔥

| 级别 | 脏读 | 不可重复读 | 幻读 |
|:---|:---:|:---:|:---:|
| 读未提交 | ✅ | ✅ | ✅ |
| **读已提交** | ❌ | ✅ | ✅ |
| **可重复读**（InnoDB默认）🔥 | ❌ | ❌ | ✅(MVCC解决部分) |
| 可串行化 | ❌ | ❌ | ❌ |

---

## 第4章 锁机制 🔥🔥

### 4.1 锁分类

| 维度 | 分类 | 说明 |
|:---|:---|:---|
| 粒度 | 表锁 / 行锁 / 间隙锁 | InnoDB 支持行锁 |
| 类型 | 共享锁(S) / 排他锁(X) | 读锁 / 写锁 |
| 思想 | 乐观锁 / 悲观锁 | CAS 版本号 / SELECT FOR UPDATE |

### 4.2 InnoDB 行级锁 🔥

```sql
-- 共享锁（S Lock）—— 其他事务可读不可写
SELECT * FROM user_info WHERE id = 1 LOCK IN SHARE MODE;

-- 排他锁（X Lock）—— 其他事务不可读不可写
SELECT * FROM user_info WHERE id = 1 FOR UPDATE;

-- 间隙锁（Gap Lock）—— 防止幻读
-- 锁定范围而非行，例如 id 在 (5, 10) 之间的间隙
```

### 4.3 死锁处理

```sql
-- 查看死锁日志
SHOW ENGINE INNODB STATUS\G

-- InnoDB 自动检测死锁，回滚代价较小的事务
-- 预防措施：
-- 1. 按固定顺序访问表和行
-- 2. 事务尽量短小
-- 3. 合理使用索引（避免锁升级为表锁）
```

---

## 第5章 MVCC 原理 🔥🔥

### 5.1 什么是 MVCC

MVCC（Multi-Version Concurrency Control）多版本并发控制，通过保存数据的历史版本，实现**读写不冲突**。

### 5.2 实现机制

```
每行记录隐藏列:
┌──────────────┬─────────────────┬──────────────────┐
│ DB_TRX_ID    │ DB_ROLL_PTR     │ DB_ROW_ID        │
│ (事务ID)      │ (回滚指针)       │ (行ID)           │
└──────┬───────┴────────┬────────┴──────────────────┘
       │                │
       ▼                ▼
  最近修改的事务ID    指向 Undo Log 中的旧版本
```

### 5.3 ReadView 🔥

```
ReadView（快照读时创建）：
- m_ids：当前活跃的事务ID列表
- min_trx_id：最小活跃事务ID
- max_trx_id：下一个待分配事务ID
- creator_trx_id：创建该 ReadView 的事务ID

可见性判断：
1. trx_id == creator_trx_id → 可见（自己修改的）
2. trx_id < min_trx_id → 可见（已提交）
3. trx_id >= max_trx_id → 不可见（未来事务）
4. trx_id in m_ids → 不可见（未提交）
5. trx_id not in m_ids → 可见（已提交）
```

**RC vs RR 的区别**：
- **RC（读已提交）**：每次 SELECT 都创建新的 ReadView
- **RR（可重复读）**：只在第一次 SELECT 时创建 ReadView 🔥

---

## 第6章 日志系统 ⭐

| 日志 | 说明 | 作用 |
|:---|:---|:---|
| **Redo Log** 🔥 | InnoDB 引擎层，物理日志 | 崩溃恢复（持久性） |
| **Undo Log** | InnoDB 引擎层，逻辑日志 | 事务回滚 + MVCC |
| **Binlog** 🔥 | Server 层，逻辑日志 | 主从复制、数据恢复 |

### 更新语句执行流程 🔥

```
UPDATE user_info SET age = 26 WHERE id = 1;

1. 从 Buffer Pool 读取 id=1 的数据页（不在则从磁盘加载）
2. 写入 Undo Log（记录旧值，用于回滚）
3. 更新 Buffer Pool 中的数据页
4. 写入 Redo Log（prepare 状态）
5. 写入 Binlog
6. 提交事务，Redo Log 改为 commit 状态（两阶段提交）
```

---

## 第7章 面试题 🔥🔥🔥

### Q1：B+Tree 和 B-Tree 的区别？
> B+Tree 数据只在叶子节点，叶子节点有双向链表。B-Tree 所有节点都存数据。B+Tree 范围查询更快，更适合数据库。

### Q2：聚簇索引和非聚簇索引？
> 聚簇索引：叶子节点存整行数据（主键索引）。非聚簇索引：叶子节点存主键值（需要回表查询）。

### Q3：MySQL 默认隔离级别？如何解决幻读？
> 可重复读（Repeatable Read）。通过 **MVCC + 间隙锁（Gap Lock）** 解决幻读。

### Q4：如何优化慢 SQL？
> 1. `EXPLAIN` 分析执行计划
> 2. 添加合适的索引
> 3. 避免索引失效
> 4. 避免 `SELECT *`
> 5. 分页优化、子查询优化

### Q5：MVCC 的原理？
> 通过 Undo Log 保存历史版本，ReadView 判断可见性。RR 级别下第一次 SELECT 创建 ReadView，后续读取相同快照。RC 级别每次 SELECT 都创建新 ReadView。

### Q6：Redo Log 和 Binlog 的区别？
> | 对比 | Redo Log | Binlog |
> |:---|:---|:---|
> | 层级 | InnoDB 引擎层 | Server 层 |
> | 类型 | 物理日志（数据页修改） | 逻辑日志（SQL语句/行变更） |
> | 写入 | 循环覆盖（固定大小） | 追加写入（不覆盖） |
> | 作用 | 崩溃恢复（持久性） | 主从复制、数据恢复 |

### Q7：什么是两阶段提交？为什么需要？
> Redo Log 先写 prepare，再写 Binlog，最后 Redo Log 写 commit。保证 Redo Log 和 Binlog 数据一致性。如果中间崩溃，可根据 Binlog 判断是否需要回滚。

### Q8：索引覆盖和索引下推是什么？
> **索引覆盖**：查询的列全在索引中，无需回表。
> **索引下推（ICP）**：在索引遍历时提前过滤不满足条件的行，减少回表次数。MySQL 5.6+ 支持。
