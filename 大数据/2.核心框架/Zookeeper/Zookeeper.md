# Zookeeper

> 🔗 **官方网站**：https://zookeeper.apache.org/
> 📖 **官方文档**：https://zookeeper.apache.org/doc/current/
> 📌 **学习版本**：ZooKeeper 3.7.x

---

---

## 第1章 Zookeeper 概述 ⭐

### 1.1 什么是 Zookeeper

ZooKeeper 是一个开源的**分布式协调服务**，为分布式应用提供一致性服务，包括：配置管理、命名服务、分布式锁、集群管理等。

### 1.2 核心特点

| 特点 | 说明 |
|:---|:---|
| **顺序一致性** | 来自同一客户端的请求按顺序执行 |
| **原子性** | 更新要么全部成功，要么全部失败 |
| **单一视图** | 无论连接哪个节点，看到的数据一致 |
| **可靠性** | 数据一旦更新成功就持久化 |
| **及时性** | 客户端在一定时间内能获取最新数据 |

### 1.3 数据结构 🔥

ZooKeeper 的数据模型是**树形结构**（类似文件系统），每个节点称为 **ZNode**。

```
/
├── /hadoop
│   ├── /hadoop/namenode
│   └── /hadoop/datanode
├── /kafka
│   ├── /kafka/brokers
│   └── /kafka/consumers
└── /hbase
    └── /hbase/master
```

每个 ZNode 最多存储 **1MB** 数据。

### 1.4 ZNode 类型 🔥

| 类型 | 说明 | 应用 |
|:---|:---|:---|
| **持久节点**（Persistent） | 客户端断开后节点仍存在 | 配置信息 |
| **临时节点**（Ephemeral） | 客户端断开后节点**自动删除** | 服务注册/发现 |
| **持久顺序节点** | 节点名后自动追加递增序号 | 全局ID生成 |
| **临时顺序节点** | 临时 + 顺序 | **分布式锁** |

---

## 第2章 选举机制 🔥🔥🔥

> **ZK 选举机制是面试必问考点！**

### 2.1 第一次启动选举

假设有 5 台服务器（SID 1~5）依次启动：

```
1. Server1 启动 → 投自己一票 → 无法获得半数以上 → 等待
2. Server2 启动 → 投自己一票 → 与Server1交换选票
   → Server2 的 SID 大 → Server1 改投 Server2
   → Server2 得 2 票，仍未过半 → 等待
3. Server3 启动 → 投自己一票 → 交换选票
   → Server3 的 SID 最大 → Server1、2 改投 Server3
   → Server3 得 3 票 > 半数(5/2=2.5) → ✅ Server3 当选 Leader
4. Server4 启动 → 已有 Leader → 成为 Follower
5. Server5 启动 → 已有 Leader → 成为 Follower
```

**选举规则**：**优先比较 ZXID（事务ID），ZXID 大的优先；ZXID 相同比较 SID（服务器ID），SID 大的优先**

### 2.2 非第一次选举（Leader 挂了） 🔥

```
1. Follower 发现无法与 Leader 通信
2. 变更为 LOOKING 状态，开始新一轮选举
3. 每台服务器发出投票：(SID, ZXID)
4. 收到其他投票后比较：
   - 先比 ZXID，大的胜出
   - ZXID 相同比 SID，大的胜出
5. 超过半数服务器同意 → 新 Leader 产生
```

---

## 第3章 监听器原理 🔥🔥

### 3.1 Watcher 机制

```
┌──────────┐    1.注册Watcher    ┌──────────┐
│  Client  │ ──────────────────▶ │   ZK     │
│          │                     │  Server  │
│          │ ◀────────────────── │          │
└──────────┘    2.触发通知        └──────────┘
```

**特点**：
- **一次性触发**：Watcher 触发后失效，需要重新注册
- **轻量级**：通知只告知"发生了变化"，不传递变化内容
- **异步**：通知通过异步回调

### 3.2 常见监听

| 监听类型 | 方法 | 说明 |
|:---|:---|:---|
| 节点数据变化 | `getData(path, watch)` | 节点内容修改时触发 |
| 子节点变化 | `getChildren(path, watch)` | 子节点增减时触发 |
| 节点创建/删除 | `exists(path, watch)` | 节点创建或删除时触发 |

---

## 第4章 客户端命令 ⭐

```bash
# 连接
bin/zkCli.sh -server hadoop102:2181

# 查看
ls /                         # 列出子节点
ls -s /                      # 带详细信息
get /node1                   # 获取节点数据
stat /node1                  # 获取节点状态

# 创建
create /node1 "hello"        # 持久节点
create -e /node2 "temp"      # 临时节点
create -s /node3 "seq"       # 顺序节点
create -e -s /node4 "es"     # 临时顺序节点

# 修改
set /node1 "world"

# 删除
delete /node1                # 删除（无子节点）
deleteall /parent            # 递归删除

# 监听
get -w /node1                # 监听节点数据变化
ls -w /                      # 监听子节点变化
```

---

## 第5章 Java API 🔥

```java
import org.apache.zookeeper.*;
import java.util.List;
import java.util.concurrent.CountDownLatch;

public class ZkClient {
    private static final String CONNECT_STRING = "hadoop102:2181,hadoop103:2181,hadoop104:2181";
    private static final int SESSION_TIMEOUT = 2000;
    private static ZooKeeper zk;
    private static CountDownLatch latch = new CountDownLatch(1);

    public static void main(String[] args) throws Exception {
        // 1. 创建连接
        zk = new ZooKeeper(CONNECT_STRING, SESSION_TIMEOUT, event -> {
            if (event.getState() == Watcher.Event.KeeperState.SyncConnected) {
                latch.countDown();
            }
            // 监听回调
            System.out.println("事件类型: " + event.getType() + ", 路径: " + event.getPath());
        });
        latch.await();
        
        // 2. 创建节点
        String path = zk.create("/atguigu", "hello".getBytes(),
                ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        System.out.println("创建节点: " + path);

        // 3. 获取子节点并监听
        List<String> children = zk.getChildren("/", true);
        System.out.println("子节点: " + children);

        // 4. 获取数据
        byte[] data = zk.getData("/atguigu", false, null);
        System.out.println("数据: " + new String(data));

        // 5. 修改数据
        zk.setData("/atguigu", "world".getBytes(), -1);

        // 6. 判断节点是否存在
        if (zk.exists("/atguigu", false) != null) {
            System.out.println("节点存在");
        }

        // 7. 删除节点
        zk.delete("/atguigu", -1);

        zk.close();
    }
}
```

---

## 第6章 分布式锁 🔥

### 6.1 实现原理

```
1. 所有客户端在锁节点 /locks 下创建临时顺序节点
2. 判断自己是否为序号最小的节点
   - 是 → 获取锁
   - 否 → 监听前一个节点
3. 释放锁：删除自己的节点
4. 前一个节点删除 → 下一个节点收到通知 → 获取锁
```

### 6.2 Curator 框架（推荐） 🔥

```java
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.curator.retry.ExponentialBackoffRetry;

public class CuratorLockDemo {
    public static void main(String[] args) {
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString("hadoop102:2181,hadoop103:2181,hadoop104:2181")
                .sessionTimeoutMs(2000)
                .retryPolicy(new ExponentialBackoffRetry(1000, 3))
                .build();
        client.start();

        // 分布式可重入锁
        InterProcessMutex lock = new InterProcessMutex(client, "/locks");

        new Thread(() -> {
            try {
                lock.acquire();
                System.out.println(Thread.currentThread().getName() + " 获取锁");
                Thread.sleep(3000);
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                try { lock.release(); } catch (Exception e) { e.printStackTrace(); }
            }
        }, "Thread-1").start();
    }
}
```

---

## 第7章 ZooKeeper 在大数据中的应用 ⭐

| 框架 | ZK 作用 |
|:---|:---|
| **Hadoop** | NameNode HA 主备切换 |
| **HBase** | Master 选举、RegionServer 存活监控、Meta 表定位 |
| **Kafka** | Broker 注册、Topic/Partition 元数据管理（新版 KRaft 去 ZK） |
| **Flink** | 集群 HA、JobManager 选举 |

---

## 第8章 高频面试题 🔥🔥🔥

### Q1：ZooKeeper 的选举机制？
> 先比 ZXID（事务ID），大的优先；ZXID 相同比 SID（服务器ID），大的优先。超过半数同意即当选。

### Q2：ZooKeeper 的监听器原理？
> 客户端注册 Watcher 到 ZK 服务端，当监听的节点发生变化时，服务端通知客户端。**一次性触发**，需要重新注册。

### Q3：ZK 集群为什么推荐奇数台？
> 半数机制要求存活节点 > 总节点/2。3 台容忍挂 1 台，4 台也只能容忍挂 1 台，所以 3 台和 4 台容错能力相同，奇数台更节省资源。

### Q4：ZK 的临时节点有什么作用？
> 客户端断开连接后自动删除。应用：服务注册/发现、分布式锁、集群存活监控。
