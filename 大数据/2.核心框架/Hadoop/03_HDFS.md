# HDFS

> 🔗 **官方网站**：https://hadoop.apache.org/
> 📖 **HDFS 文档**：https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html
> 📌 **学习版本**：Hadoop 3.3.x

---

---

## 第1章 HDFS 概述 ⭐

### 1.1 什么是 HDFS

HDFS（Hadoop Distributed File System）是 Hadoop 的**分布式文件系统**，设计用于在廉价硬件上存储超大数据集。

### 1.2 HDFS 架构 🔥🔥

![HDFS架构](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/images/hdfsarchitecture.png ':size=700')

| 组件 | 说明 |
|:---|:---|
| **NameNode（NN）** | 管理文件系统元数据（文件名、目录、权限、块映射） |
| **DataNode（DN）** | 存储实际的数据块（Block） |
| **SecondaryNameNode（2NN）** | 辅助 NN 进行元数据合并（不是热备！） |
| **Client** | 与 NN 交互获取文件位置，与 DN 读写数据 |

### 1.3 核心参数 ⭐

| 参数 | 默认值 | 说明 |
|:---|:---|:---|
| `dfs.blocksize` | **128MB** | 数据块大小（生产常设 256MB） |
| `dfs.replication` | **3** | 副本数 |
| `dfs.namenode.name.dir` | | NN 元数据存储路径 |
| `dfs.datanode.data.dir` | | DN 数据存储路径 |

---

## 第2章 HDFS 读写流程 🔥🔥🔥

### 2.1 写流程

```
1. Client 请求 NameNode 上传文件
2. NN 检查权限和文件是否存在
3. NN 返回可以上传，以及 DataNode 列表
4. Client 将文件切分为 Block
5. Client 请求 DN1 上传 Block（建立 Pipeline: DN1→DN2→DN3）
6. DN1 收到数据后传给 DN2，DN2 传给 DN3
7. 逐一应答（DN3→DN2→DN1→Client）
8. 所有 Block 写完后，Client 通知 NN，更新元数据
```

### 2.2 读流程

```
1. Client 请求 NameNode 读取文件
2. NN 返回文件的 Block 列表及 DataNode 位置（就近原则）
3. Client 并行从最近的 DN 读取各 Block
4. Client 合并 Block 为完整文件
```

### 2.3 副本存放策略 🔥

```
第1个副本：客户端所在节点（或随机节点）
第2个副本：另一个机架的随机节点
第3个副本：第2个副本同机架的不同节点
```

---

## 第3章 NN 与 2NN 工作机制 🔥🔥

### 3.1 元数据存储机制

NameNode 的元数据存储在**内存 + 磁盘**中：

| 文件 | 说明 |
|:---|:---|
| **FsImage** | 元数据的完整镜像（磁盘快照） |
| **Edits Log** | 每次写操作追加的编辑日志 |
| **内存** | FsImage + Edits 合并后的最新元数据 |

### 3.2 CheckPoint 流程 🔥

```
NameNode                    SecondaryNameNode
    │                              │
    │  ① 触发条件满足（默认1小时或100万条edits）
    │ ─────edits log──────────▶    │
    │                              │ ② 加载 fsimage + edits
    │                              │ ③ 在内存中合并
    │  ④ 新的 fsimage             │
    │ ◀──────────────────────────  │
    │ ⑤ 替换旧 fsimage            │
```

> 🔥 **核心理解**：NN 负责接收请求写 Edits，2NN 负责定期合并 FsImage + Edits，减轻 NN 启动压力。

### 3.3 DataNode 工作机制

```
DataNode 启动 → 向 NN 注册 → 周期性汇报
  │
  ├── 每 3s 心跳（Heartbeat）  → NN 确认 DN 存活
  ├── 每 6h 块报告（Block Report）→ NN 更新块映射表
  └── 超过 10min+30s 无心跳     → NN 判定 DN 死亡，开始复制副本
```

---

## 第4章 HDFS 常用命令 🔥

```bash
# 文件操作
hadoop fs -ls /                     # 列出目录
hadoop fs -mkdir -p /user/data      # 创建目录
hadoop fs -put local.txt /user/     # 上传文件
hadoop fs -get /user/file.txt ./    # 下载文件
hadoop fs -cat /user/file.txt       # 查看文件
hadoop fs -rm -r /user/data         # 删除目录
hadoop fs -cp /src /dst             # 复制
hadoop fs -mv /src /dst             # 移动
hadoop fs -du -s -h /user/          # 查看大小
hadoop fs -chmod 777 /user/file     # 修改权限
hadoop fs -setrep 2 /user/file      # 设置副本数

# 安全模式
hdfs dfsadmin -safemode get         # 查看安全模式
hdfs dfsadmin -safemode leave       # 退出安全模式

# 管理命令
hdfs dfsadmin -report               # 集群报告
hdfs dfsadmin -refreshNodes         # 刷新节点（服役/退役）
hdfs balancer                       # 数据均衡
```

---

## 第5章 HDFS 读写性能优化 ⭐

### 5.1 小文件问题 🔥🔥

**危害**：每个文件/目录/块在 NN 占约 **150 字节**内存。1亿小文件 ≈ 15GB 内存！

**解决方案**：

| 方案 | 说明 |
|:---|:---|
| **HAR 归档** | `hadoop archive` 合并小文件为归档文件 |
| **CombineTextInputFormat** | MR 中合并小文件作为输入 |
| **Hive 合并** | `hive.merge.mapfiles=true` 自动合并 |
| **JVM 重用** | `mapreduce.job.jvm.numtasks=10` |

### 5.2 纠删码（Erasure Coding）⭐

Hadoop 3.x 新特性，用**纠删码**代替多副本：

| 对比 | 3 副本 | 纠删码 (RS-6-3) |
|:---|:---|:---|
| 存储开销 | 200% | **50%** |
| 可靠性 | 任意 2 个副本损坏可恢复 | 任意 3 个块损坏可恢复 |
| 适用 | 热数据 | **冷数据** |

```bash
# 启用纠删码策略
hdfs ec -setPolicy -path /cold_data -policy RS-6-3-1024k
```

---

## 第6章 面试题 🔥🔥🔥

### Q1：HDFS 读写流程？
> 见第2章详解。写：Client→NN→Pipeline(DN1→DN2→DN3)→应答。读：Client→NN→就近DN读取。

### Q2：HDFS 为什么块大小是 128MB？
> 如果块太小，NN 存储的元数据太多，开销大。如果块太大，传输时间长。128MB 是寻址时间和传输时间的平衡点（寻址约 10ms，传输约 1s，占比约 1%）。

### Q3：NameNode 和 SecondaryNameNode 的区别？
> NN 管理元数据。2NN 不是 NN 的热备！它的作用是**定期合并 fsimage 和 edits log**，减轻 NN 的压力。

### Q4：HDFS 如何保证数据可靠性？
> 1. 副本机制（默认 3 副本）
> 2. 心跳检测（DN 定期向 NN 汇报）
> 3. 数据校验（CRC 校验）
> 4. 数据恢复（副本不足时自动复制）

### Q5：NN 的安全模式是什么？
> NN 启动时进入安全模式，此时文件系统**只读**。等待 DN 上报块信息，当满足最小副本条件后自动退出安全模式。

### Q6：HDFS 小文件问题及解决方案？
> 小文件会导致 NN 内存压力大（每个文件约 150 字节元数据）。解决：HAR 归档、CombineTextInputFormat、Hive 合并、上游采集端合并。

### Q7：HDFS 的联邦机制（Federation）？
> 多个 NameNode 各自管理一部分命名空间，共享所有 DataNode。解决单 NN 内存瓶颈和单点故障问题。
