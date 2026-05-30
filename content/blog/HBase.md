---
title: "HBase 深入浅出"
date: 2018-08-15T15:43:54+08:00
tags: ["HBase", "Database"]
draft: false
---

## 为什么需要 HBase？

传统关系型数据库在数据量达到 TB 级、写入吞吐量达到每秒百万行时会出现瓶颈：
- MySQL 分库分表后跨节点 Join/聚合困难
- 写入吞吐受限于单机 B+Tree 的随机写性能

HBase 的解决思路：**LSM-Tree + 列式存储 + 水平扩展**。它将随机写转为顺序写（先写内存再刷盘），通过 Region 自动分裂实现水平扩展。

## 1. HBase 核心组件

### 1.1 HMaster

**源码路径**：`org.apache.hadoop.hbase.master.HMaster`

核心职责：
- **元数据管理**：处理 DDL（创建/删除/修改表），维护 `hbase:meta` 表
- **负载均衡**：监控 RegionServer 负载，分配/迁移 Region
- **故障恢复**：RegionServer 宕机时重新分配其 Region

**启动流程**：
1. 连接 ZooKeeper，尝试创建 HMaster 临时节点（选主）
2. 加载 `hbase:meta` 表，获取所有 Region 分布信息
3. 启动后台线程：LoadBalancer（定期均衡 Region）、CatalogJanitor（清理 meta 中的无效条目）

**高可用机制**：通过 ZooKeeper 选举，同一时间只有一个活跃 HMaster。Standby HMaster 监听活跃节点，发现 znode 消失后立刻尝试成为主。

**面试常见问题：HMaster 宕机后 HBase 还能工作吗？**
- 可以。读写操作由 RegionServer 直接处理，HMaster 只负责 DDL 和负载均衡。已有表的读写不受影响，但无法创建新表或修改表结构。

### 1.2 RegionServer

**源码路径**：`org.apache.hadoop.hbase.regionserver.HRegionServer`

核心职责与架构：

```
┌─────────────────── RegionServer ───────────────────┐
│  ┌────────┐  ┌────────┐  ┌────────┐                │
│  │ Region1 │  │ Region2 │  │ Region3 │  ← 每个 Region 负责一段 RowKey 范围
│  └────┬───┘  └────┬───┘  └────┬───┘                │
│       │           │           │                     │
│  ┌────▼───────────▼───────────▼─────┐               │
│  │  Store(ColumnFamily: info)       │               │
│  │  ┌────────────┐ ┌─────────────┐ │               │
│  │  │ MemStore   │ │ StoreFile   │ │               │
│  │  │(SkipList)  │ │ (HFile × N) │ │               │
│  │  └────────────┘ └─────────────┘ │               │
│  ├──────────────────────────────────┤               │
│  │ BlockCache (LruBlockCache)       │               │
│  │ WAL (Write Ahead Log → HDFS)    │               │
│  └──────────────────────────────────┘               │
└────────────────────────────────────────────────────┘
```

#### 写流程详解

```
Client → RegionServer
  1️⃣ 写入 WAL（HDFS，保证不丢数据）
  2️⃣ 写入 MemStore（ConcurrentSkipListMap，有序）
  ─── 返回成功给客户端 ───
  3️⃣ 后台 Flush 线程：当 MemStore 达到阈值（128MB）→ 生成 HFile
  4️⃣ 后台 Compaction：合并小 HFile，删除已删除/过期的数据
```

写入为什么先写 WAL 再写 MemStore？
- WAL 在 HDFS 上（多副本），RegionServer 宕机后可以通过 WAL 恢复 MemStore 中未刷盘的数据
- WAL 使用顺序写（HDFS Append），性能远高于随机写

为什么 MemStore 用 ConcurrentSkipListMap？
- 数据按 RowKey 排序存储——HFile 要求数据有序，这样可以顺序写入
- ConcurrentSkipListMap 支持 O(log N) 的插入和查找，同时线程安全

#### 读流程详解

```
Client → RegionServer
  1️⃣ 查询 BlockCache（LruBlockCache，缓存热点 HFile Block）
  2️⃣ 未命中 → 查询 MemStore（未刷盘的最新数据）
  3️⃣ 未命中 → 查询 HFile（Bloom Filter 先过滤，跳过不含该 RowKey 的文件）
  4️⃣ 合并所有结果（MemStore + 多个 HFile），按时间戳取最新版本
```

**BlockCache vs MemStore**：MemStore 存的是还没写入 HFile 的数据（写缓存），BlockCache 缓存已读过的 HFile Block（读缓存）。两者分离，避免写操作影响读缓存。

### 1.3 Region

**源码路径**：`org.apache.hadoop.hbase.regionserver.HRegion`

Region 按 RowKey 范围水平切分表数据。每个 Region 包含多个 Store（每列族一个）。

**Region 分裂流程**：

```
1. RegionServer 检测 Region 大小超过阈值 (hbase.hregion.max.filesize, 默认 10GB)
2. 将 Region 下线（短暂不可用）
3. 找到 Region 的中点 RowKey，拆分为两个子 Region
4. 在 hbase:meta 中更新 Region 信息
5. 两个子 Region 上线，客户端刷新 meta 缓存后路由到新 Region
```

分裂对读写的影响：分裂期间该 Region 短暂不可用（毫秒级），通过 WAL 和 Meta 更新保证一致性。

### 1.4 WAL（Write Ahead Log）

**源码路径**：`org.apache.hadoop.hbase.wal.WAL`

```
每次写入流程：
1. RegionServer 将写入操作封装为 WALEdit
2. 通过 FSHLog 写入 HDFS 上的 WAL 文件（顺序追加，高性能）
3. 写入成功后，再写入 MemStore
4. 当 MemStore 刷盘成功 → 该部分 WAL 可截断

崩溃恢复流程：
1. RegionServer 宕机 → HMaster 感知（通过 ZooKeeper Session 超时）
2. HMaster 将该 RS 上的 Region 重新分配给其他 RS
3. 新 RS 加载 WAL，回放数据到 MemStore
4. MemStore 刷盘后，数据恢复完成
```

**WAL 性能优化**：默认是同步写入（每条写入都 fsync），可通过 `hbase.wal.hsync=false` 改为异步（风险：RegionServer 进程级崩溃可能丢失少量数据）。

## 2. HBase 读写流程深入

### 2.1 完整写入流程（含客户端侧）

```
1. 客户端从 hbase:meta 获取 RowKey 对应的 RegionServer
2. 客户端直连该 RegionServer（无中间代理）
3. RegionServer 接收请求 → 写入 WAL（多副本到 HDFS）
4. 写入 MemStore（ConcurrentSkipListMap）
5. 返回成功给客户端
6. 异步 Flush 线程在 MemStore 达到 128MB（`hbase.hregion.memstore.flush.size`）时：
   ├─ 在窗口中保持最新写入（不阻塞）
   ├─ 将当前 MemStore 快照转换为不可变数据
   └─ 写入 HFile（按照 RowKey 顺序）
```

**数据一致性保证**：WAL（HDFS 3 副本）+ MemStore 的 Snapshot + Flush 机制确保即使 RS 宕机也不会丢数据。

### 2.2 完整读取流程

```
1. 客户端从 hbase:meta 获取 RowKey 对应的 RegionServer
2. 客户端直连该 RegionServer
3. RegionServer 创建 scanner：
   ├─ 构建读取路径：MemStore + BlockCache + 所有相关 HFile
   └─ 合并结果（按 RowKey + TimeStamp + Type 排序）
4. 返回查询结果（逐行或批量）
```

**读性能优化**：
- **Bloom Filter**：跳过不含目标 RowKey 的 HFile，减少 I/O。布隆过滤器会占用内存（每个 HFile 约 1% 额外空间），但可大幅提升随机读性能
- **BlockCache**：缓存热点 HFile Block（Block 默认 64KB），顺序读取时命中率高
- **预分区（Pre-splitting）**：建表时指定 Region 数量和 split 点，避免单 Region 热点
- **压缩**：Snappy 压缩 HFile（CPU 开销小，压缩比约 2:1），减少磁盘 I/O

## 3. 高级特性

### 3.1 协处理器（Coprocessor）

类似数据库的触发器（Observer）和存储过程（Endpoint）。

**Observer**：拦截 HBase 操作，在执行前后注入自定义逻辑。

应用场景：
- **二级索引**：Put 操作时写入索引表（Observer 拦截 postPut）
- **权限校验**：Get 操作前检查用户权限（Observer 拦截 preGet）
- **审计日志**：记录所有 DML 操作

**Endpoint**：在 RegionServer 上执行聚合计算，避免客户端拉回大量数据。

场景示例：统计某个表的总行数——不通过 Endpoint 的话需要 Scan 全表拉回客户端，通过 Endpoint 各 Region 并行统计后返回汇总结果。

### 3.2 压缩与编码

| 压缩算法 | 压缩比 | CPU 开销 | 场景 |
|---------|-------|---------|------|
| GZIP | 最高 | 最高 | 归档数据，读少 |
| Snappy | 中（~2:1） | 低 | 平衡型（热数据推荐） |
| LZO | 中 | 中 | 兼容性差（需安装 native lib） |
| ZSTD | 高 | 中 | JDK 11+，Facebook 开发 |

设置方式：
```bash
# 建表时指定
create 'mytable', { NAME => 'cf', COMPRESSION => 'SNAPPY' }
# 已有表修改
alter 'mytable', { NAME => 'cf', COMPRESSION => 'GZ' }
```

## 4. 实战排查

1. **RegionServer 内存溢出（OOM）**
   - 症状：RS 进程消失，HMaster 重新分配 Region
   - 检查 MemStore 大小：`hbase.hregion.memstore.flush.size` × Region 数量 = RS 写缓冲区总量
   - 通常原因为 Region 过多导致 MemStore 总和超过 RS 堆内存——降低 `hbase.regionserver.global.memstore.size`（默认 0.4，即 40% 堆内存）

2. **读延迟高**
   - BlockCache 命中率低：增大 `hfile.block.cache.size`（默认 0.4）
   - HFile 数量过多：触发 Major Compaction 合并（但 Major Compaction 期间 I/O 压力大）
   - Bloom Filter 未开启：建表时 `BLOOMFILTER => 'ROW'`

3. **Region 热点**
   - 原因：RowKey 设计不合理，顺序递增导致所有写请求打在一个 Region 上
   - 解决方案：RowKey 加盐（前缀 hash）——如 `MD5(id).substring(0,4) + id` 使数据均匀分布

4. **Major Compaction 影响读写**
   - Major Compaction 合并所有 HFile，I/O 消耗大
   - 建议在低峰期触发：`hbase.hregion.majorcompaction` 设置为 0（关闭自动），通过 crontab 手动触发

5. **ZooKeeper Session 超时导致 RegionServer 被误判宕机**
   - 检查 GC 日志——长时间的 Full GC 可能让 ZK 认为 RS 失联
   - 调大 `zookeeper.session.timeout`（默认 90 秒），或缩短 GC 暂停时间
