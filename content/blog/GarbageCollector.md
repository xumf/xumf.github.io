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
1. Serial GC
   * 特点：
      * 单线程垃圾回收
      * 适用于单核 CPU 或小型应用
      * 回收时会暂停所有应用线程（Stop-The-World，STW）
   * 回收流程：
      * 新生代：使用复制算法（Copying）
      * 老年代：使用标记-整理算法（Mark-Compact）
   * 适用场景：
      * 客户端应用和小型服务端应用
   * 重要参数：
      * `-XX:UseSerialGC` ：启用 Serial GC
2. Parallel GC（Throughput GC）
   * 特点：
      * 多线程垃圾回收器
      * 目标是最大化吞吐量（Throughput）
      * 回收时会暂停所有应用线程（STW）
   * 回收流程：
      * 新生代：使用复制算法（Copying）
      * 老年代：使用标记-整理算法（Mark-Compact）
   * 适用场景：
      * 多核 CPU 且对吞吐量要求高的应用
   * 重要参数：
      * `-XX:+UseParallelGC` ：启用 Parallel GC
      * `-XX:ParallelGCThreads` ：设置并行 GC 线程数
      * `-XX:MaxGCPauseMillis` ：设置最大 CG 暂停时间目标
      * `-XX:GCTimeRatio` ：设置吞吐量目标（GC 时间与总时间目标）
3. CMS GC（Concurrent Mark Sweep）
   * 特点：
      * 并发垃圾回收器，尽量减少 STW 时间
      * 适用于对延迟敏感的应用
      * 使用标记-清除算法（Mark-Sweep），会产生内存碎片
   * 回收流程：
      * 初始标记（STW）：标记 GC Roots 直接关联对象
      * 并发标记：并发标记所有可达对象
      * 重新标记（STW）：修正并发标记期间的变化
      * 并发清除：并发清除不可达对象
   * 适用场景：
      * 对延迟敏感的应用，如 Web 服务
   * 重要参数：
      * `-XX:+UseConcMarkSweepGC` ：启用 CMS GC
      * `-XX:CMSInitiatingOccupancyFraction` ：设置老年代使用率触发 CMS 的阈值（默认 68%）
      * `-XX:UseCMSCompactAtFullCollection` ：在 Full GC 时进行内存整理
      * `-XX:CMSFullGCsBeforeCompaction` ：设置多少次 Full GC 后进行内存整理
4. G1 GC（Garbage First）
   * 特点：
      * 面向服务端应用的垃圾回收器
      * 将堆内存划分为多个 Region，优先回收垃圾最多的 Region
      * 目标是平衡吞吐量和延迟
   * 回收流程：
      * 初始标记（STW）：标记 GC Roots 直接关联的对象
      * 并发标记：标记所有的可达对象
      * 最终标记（STW）：修正并发标记期间的变化
      * 筛选回收（STW）：根据回收价值排序 Region，优先回收垃圾最多的 Region
   * 适用场景：
      * 大内存、多核 CPU 的应用
      * 重要参数：
         * `-XX:+UseG1GC` ：启用 G1 GC
         * `-XX:MaxGCPauseMillis` ：设置最大 GC 暂停时间的目标（默认 200ms）
         * `-XX:G1HeapRegionSize` ：设置 G1 Region 的大小
         * `-XX:InitiatingHeapOccupancyPercent` ：设置堆使用率触发并发标记的阈值（默认 45%）
5. ZGC（Z Garbage Collector）
   * 特点：
      * 低延迟垃圾回收器，STW 时间极短（通常少于 10ms）
      * 支持最大堆内存（TB 级别）
      * 使用染色指针（Colored Pointers）和读屏障（Load Barrier）技术
   * 回收流程：

```Plain Text
1. 初始化阶段（Initial Mark）
这是 ZGC 回收的第一个阶段，主要目标是标记所有存活的根对象。此时，ZGC 会标记所有活动线程和 JVM 堆栈中的对象，以及直接引用的对象。

启动时标记：在这个阶段，ZGC 会先标记应用程序中所有的“根对象”（如活动线程的栈、直接引用的对象等）。这个过程是 Stop-the-World（STW）操作，因为它需要暂停应用程序的执行，确保能够访问到所有根对象。
彩色指针标记：在此阶段，对象会被标记为活动状态。ZGC 会使用彩色指针技术，在对象的指针低位或高位嵌入额外的标记信息，表示对象的状态（例如，是否是“活动”对象）。
2. 并发标记（Concurrent Mark）
在这一步骤中，ZGC 会并发地标记堆中的所有对象，以确定哪些对象是活动的（存活的）。

并发标记：在并发标记阶段，ZGC 会遍历堆中的所有对象，并标记所有能通过根对象访问到的对象。此阶段 与应用程序并发运行，不会暂停应用程序的执行，因此不会造成长时间的停顿。
读屏障：ZGC 使用 读屏障 技术来确保指向活动对象的指针会被正确地修正，特别是当对象被移动时。读屏障帮助 ZGC 在并发标记阶段管理对象的状态。
Marking Phase：对象通过指针传播，标记所有从根对象可达的对象。如果对象可达，它就会被标记为“存活”。
3. 并发移除（Concurrent Relocation）
ZGC 通过并发的方式，在标记阶段之后开始清理垃圾并将存活对象移动到新的内存区域。

对象移动（Relocation）：ZGC 通过使用彩色指针将对象迁移到堆的连续区域，这样可以避免堆内存碎片。此阶段仍然是并发进行的，因此并不会阻塞应用程序的执行。
空间整理：ZGC 会将存活的对象移动到新的位置，目的是为了更好地利用堆空间并减少碎片。由于对象的内存位置发生变化，ZGC 使用读屏障来确保应用程序始终访问到正确的对象位置。
4. 并发清理（Concurrent Sweep）
在 ZGC 的并发清理阶段，垃圾回收器会清理被标记为垃圾的对象。

清理未标记的对象：在这个阶段，ZGC 会清理所有标记为不可达（垃圾）的对象。这一过程会并发进行，不会阻塞应用程序线程。
标记-清理分离：ZGC 的并发清理是通过 并发清理 和 并发回收 的方式来实现的，即标记阶段和清理阶段是分开进行的。ZGC 不需要在这两者之间进行停顿。
5. 结束阶段（Final Cleanup）
当 ZGC 完成并发清理后，会进行一些最后的清理工作，准备为下一轮垃圾回收做准备。

最后整理：在这个阶段，ZGC 会对剩余的内存进行整理，将所有的存活对象放到堆内存的最前端。这有助于下一轮垃圾回收时减少碎片。
恢复应用程序：在所有并发阶段完成后，应用程序可以恢复执行，垃圾回收已经完成。这个阶段通常是 Stop-the-World 的短暂停顿。
```
   * 适用场景：
      * 对延迟极其敏感的应用，如金融交易系统
   * 重要参数：
      * `-XX:+UseZGC` ：启用 ZGC
      * `-XX:ZAllocationSpikeTolerance` ：设置分配尖峰容忍度
      * `-XX:ZCollectionInterval` ：这是 ZGC 的回收时间间隔
6. Shenandoah GC
   * 特点：
      * 低延迟垃圾回收器，STW 时间极短
      * 支持并发压缩（Concurrent Compaction）
      * 使用 Brooks 指针和读屏障技术
   * 回收流程：
      * 并发标记：并发标记所有可达对象
      * 并发转移：并发将存活对象转移到新的内存区域
      * 并发更新引用：并发更新对象引用
   * 适用场景：
      * 对延迟敏感的应用，如实时数据处理系统
   * 重要参数：
      * `-XX:+UseShenandoahGC` ：启用 Shenandoah GC
      * `-XX:ShenandoahGCHeuristics` ：设置 Shenandoah 的启发式策略
      * `-XX:ShenandoahGCMode` ：设置 Shenandoah 的工作默认

### 三、总结
|垃圾回收器|特点|适用场景|重要参数|
| ----- | ----- | ----- | ----- |
|Serial GC|单线程，STW|小型应用|\-XX:+UseSerialGC|
|Parallel GC|多线程，高吞吐量|多核 CPU，高吞吐量应用|\-XX:+UseParallelGC, -XX:ParallelGCThreads|
|CMS GC|并发，低延迟|对延迟敏感的应用|\-XX:+UseConcMarkSweepGC, -XX:CMSInitiatingOccupancyFraction|
|G1 GC|平衡吞吐量和延迟|大内存、多核 CPU 应用|\-XX:+UseG1GC, -XX:MaxGCPauseMillis|
|ZGC|极低延迟，超大堆|对延迟极其敏感的应用|\-XX:+UseZGC|
|Shenandoah GC|极低延迟，并发压缩|对延迟敏感的应用|\-XX:+UseShenandoahGC|



### 四、选择垃圾回收器的建议
* 小型应用：Serial GC。
* 高吞吐量应用：Parallel GC。
* 低延迟应用：CMS GC（JDK 8 及之前）或 G1 GC（JDK 9 及之后）。
* 超大堆内存、极低延迟应用：ZGC 或 Shenandoah GC。

通过合理选择和配置垃圾回收器，可以显著提升应用的性能和稳定性。

### 五、其他关键点
#### 1. 可达性算法
#### 2. 三色标记
#### 3. 标记清除算法
先标记，再清除

   * 优点：速度快
   * 缺点：产生碎片空间
### 4. 标记复制算法
在一块内存先标记存活对象，然后将存活对象复制到另外一块内存，最后清除原有内存

   * 优点：速度快，不产生碎片空间
   * 缺点：浪费内存空间，只能使用原有内存的一般
   * 应用：一般应用在新生代 eden，survivor1，survior2 
#### 5. 标记整理算法
先标记存活对象，将存活对象全部向一边内存迁移，最后清除另一边内存

   * 优点：不产生碎片空间
   * 缺点：速度较慢
   * 应用：CMS

### 安全点（Safe Point）
### 安全区域（Safe Area）
### 跨代引用解决方案
##### 1. G1的解决方案：记忆集（RSet）
每个 Region 维护一个 **RSet**（Remembered Set），用于记录 **其他 Region 中的对象对本 Region 的引用**。例如：

   * Region A（老年代）中的对象引用了 Region B（新生代）中的对象，则 Region B 中 RSet 会记录 Region A 中存在这种引用
**RSet 的结构**
   * RSet 本质上是一个哈希表，键是引用来源的 Region 地址，值是来源 Region 中的卡表索引（Card Index）
   * 卡表（Card Table）：每个 Region 被划分为多个卡页（Card Page，默认 512 字节），卡表是一个字节数组，每个元素对应一个卡页。当卡页内的对象存在跨 Region 引用时，对应的卡表元素会被标记为脏
##### 2. 写屏障（Write Barrier）维护 RSet
当程序修改对象引用（如 Obj.field = newObj）时，G1 通过写屏障触发一下操作：

   1. 判断是否为跨 Region 引用：检查 `obj` 和`newObj` 是否属于不同 Region
   2. 更新 RSet：如果跨 Region，将引用来源（`obj` 所在的 Region 和卡页）记录到`newObj` 所在 Region 的 RSet 中

伪代码示例：

```java
void writeBarrier(Field field, Object newObj) {
    Object oldObj = field.get();
    if (oldObj != null && isCrossRegionRef(oldObj, newObj)) {
        // 记录引用来源到目标 Region 的 RSet
        addToRSet(oldObj.region(), newObj.region(), getCardIndex(oldObj));
    }
    field.set(newObj);
}
```
##### 3. 垃圾回收处理跨代引用
以 Young GC 为例（清理新生代 Region）：

   1. 确定存活对象：通过 GC Roots 扫描新生代对象
   2. 处理跨代引用：
      1. 遍历新生代每个 Region 的 RSet，找到所有老年代 Region 对新生代的引用
      2. 将这里老年代 Region 中的卡页标记为“需要扫描”
      3. 仅扫描这些脏卡页中的对象，避免扫描整个老年代

**优势**

   * **减少扫描范围**：仅处理与新生代相关的跨代引用，而非全堆扫描
   * **并行处理**：RSet 的维护和扫描可以并行化，降低停顿时间
##### 4. RSet 的优化
为了减少内存和计算开销，G1 对 RSet 进行了优化：

   1. **稀疏哈希表**：仅在存在跨 Region 引用时分配存储空间
   2. **粒度控制**：卡页大小（512 字节）平衡了进度与内存占用
   3. **并发更新**：写屏障的操作是并发执行的，减少对应用线程阻塞
##### 5. 与其他回收器的对比
|机制|G1|CMS|
| ----- | ----- | ----- |
|**跨代引用处理**|每个 Region 维护 RSet|全局卡表（Card Table）|
|**精度**|精确到 Region 和卡页|粗粒度（卡表页）|
|**内存开销**|较高（每个 Region 一个 RSet）|较低（单个全局卡表）|
|**扫描效率**|更高效（仅扫描相关 Region）|需扫描整个老年代的脏卡页|







