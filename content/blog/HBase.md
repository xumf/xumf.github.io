---
title: "HBase 深入浅出"
date: 2018-08-15T15:43:54+08:00
tags: ["HBase", "Database"]
draft: false
---

## 1. HBase 核心组件源码深度解析
### 1.1 HMaster
**源码路径**：`org.apache.hadoop.hbase.master.HMaster` 

* 核心职责：
   * 元数据管理：
      * 管理表创建、删除、修改等 DDL 操作
      * 维护 `hbase:meta` 表，记录所有 Region 的分布信息
   * 负载均衡：
      * 监控 RegionServer 的状态，分配 Region 到 RegionServer
      * 处理 RegionServer 的故障恢复
   * 协调分裂与合并
      * 当 Region 大小超过阈值时，触发 Region 分裂
      * 合并小 Region，减少元数据开销

**源码关键点**：

* 启动流程：
   * 初始化 ZooKeeper 链接，注册 HMaster 节点
   * 加载 `hbase:meta` 表，获取所有 Region 的分布信息
   * 启动后台线程，监控 RegionServer 状态
* Region 分配
   * 通过 AssignmentManager 将 Region 分配到 RegionServer
   * 使用 ZooKeeper 协调 Region 的分配和迁移

**面试题**：

* HMaster 宕机后，HBase 还能正常工作吗？
   * 可以，HBase 的读写操作有 RegionServer 直接处理，HMaster 宕机不会影响已有表的读写，但会影响表的元数据操作（如创建表）
* HMaster 如何实现高可用？
   * 通过 ZooKeeper 选举机制，确保只有一个活跃的 HMaster

### 1.2 RegionServer
**源码路径**：`org.apache.hadoop.hbase.regionserver.HRegionServer`

* 核心职责：
   * 数据存储：
      * 管理多个 Region，每个 Region 负责表的一部分数据
      * 将数据写入 MemStore，定期刷写到 HFile
   * 读写请求处理：
      * 处理客户端的读写请求
      * 使用 BlockCache 缓存热点数据，提升读性能
   * WAL 管理
      * 每次写入数据时，先写入 WAL，确保数据可靠性
      * 定期滚动 WAL 文件，防止文件过大

**源码关键点**：

* 写流程：
   * 数据先写入 WAL，再写入 MemStore
   * 当 MemStore 达到阈值时，触发刷写（Flush）操作，将数据写如 HFile
* 读流程：
   * 首先从 BlockCache 中查找数据
   * 如果未命中，则从 MemStore 和 HFile 中读取数据
* Compaction
   * 定期合并 HFile，减少文件数量，提升读性能

**面试题**：

* RegionServer 的架构时怎么样的？
   * RegionServer 包含多个 Region，每个 Region 负责表的一部分数据。RegionServer 还包含 MemStore（内存存储）、BlockCache（读缓存）和 WAL（预写日志）
   * MemStore 的作用时什么？
      * MemStore 是内存中的数据结构，用于缓存写入的数据。当 MemStore 达到一定大小时，数据会被刷到 HFile 中

### 1.3 Region
**源码路径**：`org.apache.hadoop.hbase.regionserver.HRegion`

* 核心职责：
   * 数据分区：
      * 按行键范围划分数据
   * 数据存储：
      * 每个 Region 包含多个 Store，每个 Store 对应一个列族
      * Store 包含一个MemStore 和多个 HFile
   * 读写请求处理：
      * 处理客户端对当前 Region 的读写请求

**源码关键点：**

* Region 分裂：
   * 当 Region 大小超过阈值时，触发分裂操作
   * 分裂过程由 RegionServer 触发，HMaster 负责协调
* Store 管理：
   * 每个 Store 对应一个列族，包含一个 MemStore 和多个 HFile
   * 定期刷写 MemStore 到 HFile，合并 HFile

**面试题**：

* Region 时如何分裂的？
   * 当 Region 的大小达到阈值（默认 10GB），Region 会分裂为两个子 Region。分裂过程由 RegionServer 触发，HMaster 负责协调
* Region 分裂对读写有什么影响
   * 分裂过程中，Region 会暂时不可用，但分裂完成后，读写请求会路由到新的 Region

### 1.4 WAL（Write Ahead log）
**源码路径**：`org.apache.hadoop.hbase.wal.WAL` 

* 核心职责
   * 数据可靠性：
      * 每次写入数据时，先写入 WAL，再写入 MemStore
      * 如果 RegionServer  宕机，可以通过 WAL 恢复未刷写到 HFile 的数据
   * 日志滚动：
      * 定期滚动 WAL 文件，防止文件过大

**源码关键点**：

* WAL 写入：
   * 使用 **FSHLog**** **类将日志写入 HDFS
   * 支持批量写入和异步写入，提升性能
* WAL 恢复：
   * 当 RegionServer 重启时，通过 WAL 恢复未持久化的数据

**面试题**：

* WAL 的作用时什么？
   * WAL 记录每次写入操作，确保再 RegionServer 宕机时可以通过 WAL 恢复未持久化的数据。
* WAL 的写入性能如何优化？
   * 可以通过批量写入、异步写入等方式优化 WAL 的性能。

## 2. HBase 读写流程源码深度解析
### 2.1 写流程
1. 客户端向 RegionServer 发送写请求
2. RegionServer 将数据写入 WAL（Write Ahead Log）
3. 数据写入 MemStore
4. 当 MemStore 达到一定大小时，数据刷写到 HFile

**源码关键点**：

* WAL 写入
   * 使用 `FSHLog` 类将日志写入 HDFS
* MemStore 写入：
   * 数据写入`ConcurrentSkipListMap` ，支持高效的内存存储
* 刷写（Flush）
   * 当 MemStore 大小超过阈值时，触发刷写操作，将数据写入 HFile

**面试题：**

* **写流程中如何保证数据一致性？**
   * 通过 WAL 和 HDFS 的多副本机制保证数据一致性。

### 2.2 读流程
1. 客户端向 RegionServer 发送请求
2. RegionServer 首先从 BlockCache 中查找数据
3. 如果 BlockCache 未命中，则从 MemStore 和 HFile 中读取数据
4. 返回查询结果

**源码关键点**：

* BlockCache：
   * 使用 `LruBlockCache` 缓存热点数据
* HFile 读取：
   * 使用 `HFileReader` 读取 HFile 中的数据

**面试题：**

* 如何优化 HBase 的读性能？
   * 可以通过增大 BlockCache、使用 Bloom Filter、预分区等方式优化读性能



## 3. HBase 高级特性源码深度解析
### 3.1 协处理器（Coprocessor）
**源码路径**：`org.apache.hadoop.hbase.coprocessor` 

* 核心职责：
   * Observer
      * 拦截 HBase 的操作（如 `put`、`get`），执行自定义逻辑
   * Endpoint：
      * 在 RegionServer 上执行自定义逻辑，类似于存储过程

**源码关键点**：

* Observer：
   * 通过 `RegionObserver` 拦截 Region 的操作
* Endpoint：
   * 通过 `CoprocessorService` 实现自定义 RPC 服务

**面试题**：

* 协处理器的作用时什么？
   * 协处理器允许在 RegionServer 上执行自定义代码，用于实现二级索引、聚合计算等功能
* Observer 和 Endpoint 的区别时什么？
   * Observer 用于拦截 HBase 的操作，Enpoint 用于执行自定义逻辑

### 3.2 数据压缩与编码
**源码路径**： `org.apache.hadoop.hbase.io.compress` 、`org.apache.hadoop.hbase.io.encoding` 

* 核心职责：
   * 压缩：
      * 减少存储空间和 I/O 开销
   * 编码：
      * 优化数据存储格式，提升查询性能

**源码关键点**：

* 压缩算法：
   * 支持 GZIP、Snappy、LZO 等压缩算法
* 编码方式：
   * 支持 Diff、Prefix 等编码方式

**面试题**：

* HBase 支持哪些压缩算法？
   * HBase 支持 GZIP、Snappy、LZO 等压缩算法
* 数据编码的作用时什么？
   * 数据编码可以减少存储空间和 I/O 开销，提供查询性能