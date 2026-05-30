---
title: "垃圾回收器"
date: 2018-07-18T15:43:54+08:00
tags: ["GC", "Java", "JVM"]
draft: false
---

## 一、JVM 垃圾回收器概述

常见的垃圾回收器：

1. Serial GC
2. Parallel GC（Throughput GC）
3. CMS GC（Concurrent Mark Sweep）
4. G1 GC（Garbage First）
5. ZGC（Z Garbage Collector）
6. Shenandoah GC

## 二、垃圾回收器的特点与回收流程

### 衡量 GC 优劣的三个核心指标

- **吞吐量（Throughput）**：用户代码运行时间 /（用户代码运行时间 + GC 时间）。越高越好。
- **暂停时间（Pause Time / Latency）**：GC 时应用线程暂停的时间。越短越好。
- **内存占用（Footprint）**：GC 额外消耗的内存（如 G1 的 RSet、ZGC 的染色指针）。越少越好。

这三者不可能同时优化——任何 GC 都在三者间做取舍。

### 1. Serial GC

- **特点**：单线程，回收时全部 STW
- **为什么存在**：单核 CPU 或小型应用场景，多线程反而有额外开销
- **回收流程**：
  - 新生代：复制算法（Copying）
  - 老年代：标记-整理算法（Mark-Compact）
- **适用场景**：客户端应用、小型服务端、单核环境
- **参数**：`-XX:+UseSerialGC`
- **何时选它**：堆小于 100MB、单核 CPU。其他情况不推荐。

### 2. Parallel GC（Throughput GC）

- **特点**：多线程，目标是最大化吞吐量，但 STW 时间较长
- **设计目标**："只要暂停时间在可接受范围内，吞吐量越高越好"
- **回收流程**：
  - 新生代：复制算法（多线程）
  - 老年代：标记-整理算法（多线程）
- **适用场景**：多核 CPU、对延迟要求不高的批处理/数据分析任务
- **重要参数**：
  - `-XX:+UseParallelGC`：启用
  - `-XX:ParallelGCThreads`：并行 GC 线程数（默认 = CPU 核心数）
  - `-XX:MaxGCPauseMillis`：期望最大暂停时间（不是硬保证，默认无）
  - `-XX:GCTimeRatio`：吞吐量目标（默认 99，即允许 1% 时间花在 GC 上）

### 3. CMS GC（Concurrent Mark Sweep）

- **特点**：并发收集老年代，尽量减少 STW 时间
- **为什么选 CMS**：对于 Web 服务等延迟敏感应用，每次 Full GC 暂停几秒钟不可接受。CMS 让大部分 GC 工作与应用线程并发。
- **回收流程**：
  1. **初始标记（STW）**：标记 GC Roots 直接关联的对象——**快速**
  2. **并发标记**：从 GC Roots 开始遍历所有引用——**与应用并发**
  3. **重新标记（STW）**：修正并发标记期间引用变化导致的漏标——**较快**
  4. **并发清除**：清除不可达对象——**与应用并发**
- **核心问题**：
  - **内存碎片**：使用标记-清除算法，老年代越来越碎片化 → 最终触发 Full GC 且无法避免长时间 STW
  - **CPU 敏感**：并发阶段抢占 CPU，应用吞吐量下降
  - **浮动垃圾**：并发标记期间新产生的垃圾本次无法回收
  - **Promotion Failed**：新生代对象晋升老年代时无连续空间 → Full GC
- **重要参数**：
  - `-XX:+UseConcMarkSweepGC`：启用（JDK 9 后已废弃，JDK 14 移除）
  - `-XX:CMSInitiatingOccupancyFraction`：触发 CMS 的老年代使用率阈值（默认 68%）
  - `-XX:+UseCMSCompactAtFullCollection`：Full GC 时整理内存
  - `-XX:CMSFullGCsBeforeCompaction`：多少次 Full GC 后进行一次整理
- **为什么被废弃**：G1 在延迟和吞吐量之间的平衡表现更好，且 CMS 无法解决碎片化和浮动垃圾问题。

### 4. G1 GC（Garbage First）

- **特点**：面向大堆、多核的服务端 GC。将堆划分为多个 Region，优先回收垃圾最多的 Region（"Garbage First" 名字的由来）
- **设计思路**：不要求一次回收整个堆，而是分批次回收每个 Region，通过可预测的 pause time 模型控制停顿
- **回收流程**：

```
初始标记(STW) → 并发标记 → 最终标记(STW) → 筛选回收(STW)
```
  - **初始标记**：标记 GC Roots + 修改 TAMS（Next Top at Mark Start）指针
  - **并发标记**：从 GC Roots 遍历所有对象，记录引用变更到 SATB（Snapshot At The Beginning）队列
  - **最终标记**：处理 SATB 队列中的漏标对象
  - **筛选回收**：计算每个 Region 的回收价值（回收能释放多少空间 × 回收耗时），优先回收"垃圾最多、回收最快"的 Region

- **关键内部机制**：

| 机制 | 作用 |
|------|------|
| **Region** | 堆划分为 1MB~32MB 的 Region，每个 Region 可扮演 Eden/Survivor/Old/Humongous |
| **RSet（Remembered Set）** | 记录其他 Region 对本 Region 的引用——避免全堆扫描 |
| **SATB（Snapshot At The Beginning）** | 并发标记开始时拍堆快照，所有在标记期间变更的引用都记录到 SATB 队列 |
| **写屏障** | 对象引用变更时触发，更新 RSet + 记录 SATB |

- **适用场景**：堆大小 4GB+，多核 CPU，延迟要求 100-300ms
- **重要参数**：
  - `-XX:+UseG1GC`：启用（JDK 9+ 默认）
  - `-XX:MaxGCPauseMillis`：期望暂停时间目标（默认 200ms，G1 会努力但无法硬保证）
  - `-XX:G1HeapRegionSize`：Region 大小（默认堆/2048，1~32MB）
  - `-XX:InitiatingHeapOccupancyPercent`：触发并发标记的堆使用率（默认 45%）
  - `-XX:G1NewSizePercent`：新生代最小占比（默认 5%）
  - `-XX:G1MaxNewSizePercent`：新生代最大占比（默认 60%）

- **何时选 G1**：JDK 11+、堆 4GB+、延迟要求几百毫秒，G1 是默认选择。

### 5. ZGC（Z Garbage Collector）

- **特点**：极低延迟（STW < 10ms），与堆大小基本无关（即使是 TB 级堆）
- **为什么能这么低延迟**：几乎所有阶段都并发，只在极少数点 STW（且 STW 阶段不随堆大小增长）
- **关键技术**：

| 技术 | 作用 |
|------|------|
| **染色指针（Colored Pointers）** | 在 64 位指针的高位（42 位之后）存储 GC 状态信息（Finalizable/Remapped/Mark1/Mark0），无需额外内存 |
| **读屏障（Load Barrier）** | 每次从堆读取引用时检查指针颜色，如果对象被移动，读屏障在返回前修正引用 |
| **并发迁移（Relocation）** | 存活对象在应用运行时被迁移到新区域，旧区域清空后回收 |

- **回收流程**：
  1. **并发标记（Concurrent Mark）**：遍历对象图，标记存活对象。与 G1 不同，ZGC 的标记完全并发（初始标记的 STW 极短）。
  2. **并发迁移预备（Concurrent Prepare for Relocation）**：确定需要清理的 Region。
  3. **并发迁移（Concurrent Relocate）**：将存活对象复制到新 Region，更新引用。读屏障确保应用线程始终访问到正确地址。
  4. **并发重映射（Concurrent Remap）**：修正所有指向旧位置的引用。

- **适用场景**：TB 级堆、延迟要求 < 10ms 的金融交易/实时系统
- **重要参数**：
  - `-XX:+UseZGC`：启用（JDK 11 实验性，JDK 15 正式，JDK 18+ 默认）
  - `-XX:ZAllocationSpikeTolerance`：分配尖峰容忍度（默认 2.0）
  - `-XX:ZCollectionInterval`：最大回收间隔（秒）
- **注意事项**：ZGC 的吞吐量略低于 G1（因为读屏障开销），内存占用也更高。选 ZGC 时确认你能接受这些 Trade-off。

### 6. Shenandoah GC

- **特点**：与 ZGC 类似，目标是极低延迟。与 ZGC 的区别在于实现方式：
  - ZGC 使用染色指针（需要 64 位系统 + 特定 CPU 特性）
  - Shenandoah 使用 Brooks 指针（对象头中加一个转发指针），不依赖硬件
- **回收流程**：并发标记 → 并发转移 → 并发更新引用（比 ZGC 多一个并发更新引用阶段）
- **适用场景**：与 ZGC 类似，但不要求特定硬件
- **重要参数**：
  - `-XX:+UseShenandoahGC`：启用（JDK 12+，OpenJDK 发行版不默认包含——需要 Build 时开启）
  - `-XX:ShenandoahGCHeuristics`：启发式策略（adaptive/static/compact/aggressive）

## 三、GC 选择决策树

```
应用需要什么？
├─ 批处理/数据分析 → Parallel（要吞吐量）
├─ Web 服务/微服务
│  ├─ 堆 < 4GB → G1 或 Parallel
│  ├─ 堆 4~16GB → G1（JDK 11+ 默认）
│  ├─ 堆 > 16GB + 延迟敏感 → G1（延迟要求 < 100ms？）
│  │  └─ 是 → ZGC（JDK 17+）
│  └─ 堆 > 100GB + 极致低延迟 → ZGC
└─ 客户端/小型应用 → Serial
```

## 四、GC 优化实践

### 基础三步

1. **确定目标**：是提高吞吐量还是降低延迟？
2. **收集数据**：PrintGCDetails + GCViewer/GCEasy 分析 GC 日志
3. **调整参数**：一次只改一个参数，观察效果

### 常用 JVM 参数组合

```bash
# 打印 GC 日志（JDK 9+）
-Xlog:gc*
-Xlog:gc:gc.log

# 打印 GC 日志（JDK 8）
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log

# G1 生产环境常用参数
-XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:+PrintGCDetails

# 大堆 ZGC 常用参数
-XX:+UseZGC -Xms64G -Xmx64G -XX:ZAllocationSpikeTolerance=2.0
```

### 内存泄漏排查思路

1. Heap Dump + MAT（Memory Analyzer Tool）分析大对象
2. GC 日志中 `Full GC` 频率异常 → 确认是否为晋升失败或 MetaSpace 泄漏
3. Native Memory Tracking（NMT）检查 off-heap 内存

## 五、跨代引用与 RSet 详解

### G1 的 RSet（Remembered Set）

每个 Region 维护一个 RSet，记录其他 Region 对它的引用。为什么需要 RSet？Young GC 时只扫描新生代 Region，但老年代可能有对象引用新生代对象——有了 RSet 就不必全堆扫描。

**RSet 的结构**：哈希表，key 是来源 Region 地址，value 是来源 Region 中的卡页索引（Card Index）。卡页默认 512 字节。

**写屏障维护 RSet**：

```java
void writeBarrier(Field field, Object newObj) {
    Object oldObj = field.get();
    if (oldObj != null && isCrossRegionRef(oldObj, newObj)) {
        addToRSet(oldObj.region(), newObj.region(), getCardIndex(oldObj));
    }
    field.set(newObj);
}
```

### G1 vs CMS 跨代引用处理对比

| 机制 | G1 | CMS |
|------|-----|-----|
| 跨代引用处理 | 每个 Region 维护 RSet | 全局卡表（Card Table） |
| 精度 | 精确到 Region + 卡页 | 粗粒度（卡表页） |
| 内存开销 | 较高（每个 Region 一个 RSet） | 较低（全局卡表） |
| 扫描效率 | 仅扫描相关 Region | 需扫描整个老年代脏卡页 |
