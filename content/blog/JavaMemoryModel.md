---
title : "JMM"
date : "2021-07-09T23:36:22+08:00"
description : "Java Memory Model"
tags : [
  "Java",
  "JVM"
]

---

### 1：基本操作流程（单线程）

```mermaid
sequenceDiagram
    participant 主内存
    participant 工作内存
    participant 线程

    Note over 主内存,工作内存: 读取变量（主内存 → 工作内存）
    线程 ->> 主内存: read(var)
    主内存 -->> 工作内存: 传输变量值
    线程 ->> 工作内存: load(var)

    Note over 工作内存: 修改变量（工作内存内操作）
    线程 ->> 工作内存: assign(var, newValue)

    Note over 工作内存,主内存: 写回变量（工作内存 → 主内存）
    线程 ->> 工作内存: store(var)
    工作内存 -->> 主内存: 传输变量值
    线程 ->> 主内存: write(var)
```

### 2：多线程下的原子性问题
```mermaid
sequenceDiagram
    participant 主内存
    participant 工作内存A
    participant 线程A
    participant 工作内存B
    participant 线程B

    rect rgba(255, 0, 0, 0.1)
        Note left of 线程A: 线程A操作
        线程A ->> 主内存: read(var)
        主内存 -->> 工作内存A: var=0
        线程A ->> 工作内存A: load(var)
        线程A ->> 工作内存A: assign(var, 1)
        线程A ->> 工作内存A: store(var)
        工作内存A -->> 主内存: 准备写入var=1
    end

    rect rgba(0, 255, 0, 0.1)
        Note right of 线程B: 线程B操作（插入A的store→write之间）
        线程B ->> 主内存: read(var)（主内存仍为0）
        主内存 -->> 工作内存B: var=0
        线程B ->> 工作内存B: load(var)
        线程B ->> 工作内存B: assign(var, 1)
        线程B ->> 工作内存B: store(var)
        工作内存B -->> 主内存: 写入var=1
        主内存 -->> 线程B: write(var)完成
    end

    rect rgba(255, 0, 0, 0.1)
        Note left of 线程A: 线程A完成写入（被覆盖）
        主内存 -->> 线程A: write(var)（最终var=1，预期为2）
    end
```

### 3：同步机制下的原子性保证（如volatile）
```mermaid
sequenceDiagram
    participant 主内存
    participant 工作内存A
    participant 线程A
    participant 工作内存B
    participant 线程B

    rect rgba(0, 255, 0, 0.1)
        Note left of 线程A: 线程A操作（volatile变量）
        线程A ->> 主内存: read(var)
        主内存 -->> 工作内存A: var=0
        线程A ->> 工作内存A: load(var)
        线程A ->> 工作内存A: assign(var, 1)
        线程A ->> 工作内存A: store(var)
        工作内存A -->> 主内存: 立即写入var=1
        主内存 -->> 线程A: write(var)完成
    end

    rect rgba(0, 0, 255, 0.1)
        Note right of 线程B: 线程B操作（强制读取最新值）
        线程B ->> 主内存: read(var)（必须读取var=1）
        主内存 -->> 工作内存B: var=1
        线程B ->> 工作内存B: load(var)
    end
```
### ‌关键说明‌
1. ‌基本流程‌：
  * read→load 是主内存到工作内存的同步。
  * store→write 是工作内存到主内存的同步。
  * assign 是工作内存内的赋值操作。
2. ‌多线程原子性问题‌：
  * 未同步时，store→write 和 read→load 可能被其他线程中断，导致最终结果不符合预期。
3. 同步机制作用‌：
  * volatile 强制 store→write 和 read→load 的原子性，确保修改对其他线程立即可见。