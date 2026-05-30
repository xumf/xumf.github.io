---
title : "RabbitMQ 深入浅出"
date : "2023-03-03T20:54:29+08:00"
tags : [
  "RabbitMQ",
  "MQ"
]

---

# RabbitMQ 知识架构

## 1. AMQP 协议基础

RabbitMQ 是 AMQP 0-9-1 协议的实现，其核心模型基于**信道（Channel）**、**交换机（Exchange）**和**队列（Queue）**的协作。

### 为什么需要 Channel？

AMQP 的设计用意：每次请求都创建 TCP 连接代价太高（三次握手+慢启动），Channel 是 TCP 内的虚拟链路，多个 Channel 复用同一个 TCP 连接。每个 Channel 独立处理消息路由，支持多线程并发。

相关概念：

- **虚拟主机（VHost）**：逻辑隔离环境，每个 VHost 拥有独立的交换机、队列和权限。一个 RabbitMQ 实例可以创建多个 VHost，用于多租户隔离。
- **帧（Frame）**：AMQP 数据传输的最小单位，包含帧头、帧体和帧尾，用于封装消息和协议指令。

---

## 2. 消息流转原理

### 生产者发送消息

```
生产者 → TCP 连接 → Channel → basic.publish(exchange, routingKey, props, body) → Exchange
```

关键参数：
- `exchange`：目标交换机
- `routing_key`：路由键（交换机据此路由）
- `mandatory`：true 时消息无法路由则返回错误，false 时消息丢失
- `delivery_mode`：2 表示持久化消息

### 交换机类型

| 类型 | 路由规则 | 性能 | 场景 |
|------|---------|------|------|
| **Direct** | 精确匹配 `routing_key` = `binding_key` | 最高 | 点对点、单播 |
| **Fanout** | 广播到所有绑定队列，忽略 routing_key | 高 | 发布订阅 |
| **Topic** | 通配符匹配 `*`（一个单词）`#`（多个单词） | 中 | 按主题分类 |
| **Headers** | 根据消息 header 键值对匹配 | 低 | 复杂条件路由 |

### 消息确认机制

| 模式 | 行为 | 风险 | 场景 |
|------|------|------|------|
| **自动 ACK** | 消息发送后立即删除 | 消费者处理失败时消息丢失 | 可以容忍丢失 |
| **手动 ACK（basic.ack）** | 消费者处理完成后发送确认消息 | 无（除非消费者忘记 ACK） | 生产环境必选 |

手动 ACK 的注意事项：
- 不 ACK 也不拒绝，消息会一直留在队列中（unacked）
- 消费者断开连接后，unacked 消息重新入队（requeue），支持重试
- 拒绝时 `requeue=true` 会重新入队（可能导致死循环），`requeue=false` 进入死信队列

---

## 3. 持久化与可靠性

### 消息持久化三要素

```
队列 durable = true  +  消息 delivery_mode = 2  +  Publisher Confirm
```

三者缺一不可：
- 队列不持久化 → 队列本身在重启后消失
- 消息不持久化 → 即使队列持久化，消息也只存在内存中
- 没有 Confirm → 不知道消息是否到达 Broker

### Publisher Confirm 模式

```java
// Spring AMQP 示例: 启用 Confirm
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
    RabbitTemplate template = new RabbitTemplate(connectionFactory);
    template.setConfirmCallback((correlationData, ack, cause) -> {
        if (ack) {
            // 消息成功到达 Exchange
        } else {
            // 消息到达 Exchange 失败，记录日志/重试
        }
    });
    template.setMandatory(true);
    template.setReturnsCallback(returned -> {
        // 消息无法路由到 Queue（mandatory=true 时触发）
        log.warn("消息不可路由: {}", returned.getMessage());
    });
    return template;
}
```

Confirm 和 Return 的区别：
- **Confirm**：确认消息是否到达 Exchange
- **Return**：确认消息从 Exchange 是否路由到 Queue（mandatory=true）

---

## 4. 死信队列（DLX）

### 触发条件

1. 消息被消费者拒绝（`basic.reject` / `basic.nack`）+ `requeue=false`
2. 消息 TTL 过期（`expiration` 属性）
3. 队列达到最大长度（`x-max-length` / `x-max-length-bytes`）
4. 消息超过队列的 TTL（`x-message-ttl`）

### 应用场景

- **延迟队列**：消息设置 TTL → 过期后进入死信队列 → 消费者从死信队列消费。不需要额外插件。

### 使用示例

```java
// Spring Boot 配置延迟队列（通过死信实现）
@Bean
public Queue normalQueue() {
    Map<String, Object> args = new HashMap<>();
    args.put("x-dead-letter-exchange", "dlx.exchange");
    args.put("x-dead-letter-routing-key", "dlx.routing.key");
    args.put("x-message-ttl", 30000); // 30秒后过期
    return new Queue("normal.queue", true, false, false, args);
}

@Bean
public Queue dlq() {
    return new Queue("dlq.queue", true);
}
```

---

## 5. 集群与高可用

### 普通集群

- 所有节点共享交换机、队列元数据
- 队列数据仅存储在创建节点
- 其他节点访问该队列时内部转发（性能损耗）
- **问题**：队列所在节点宕机 → 队列数据丢失

### 镜像队列（Quorum Queue 替代）

从 RabbitMQ 3.8 起，推荐使用 **Quorum Queue** 替代传统镜像队列：

| 特性 | 镜像队列 | Quorum Queue |
|------|---------|-------------|
| 一致性协议 | GM（Guaranteed Multicast） | Raft |
| 数据模型 | Master-Slave | Leader-Follower |
| 网络分区处理 | 手动策略 | 自动（Raft 自动选主） |
| 适用场景 | 高可用 | 高可用 + 强一致（推荐） |

**生产环境推荐**：新项目直接使用 Quorum Queue。

---

## 6. 性能优化

### QoS 预取

```java
// Spring Boot: 每次只拉取 1 条消息，处理完再拉取下一条
@Bean
public SimpleRabbitListenerContainerFactory myFactory(
        ConnectionFactory connectionFactory) {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    factory.setPrefetchCount(1); // 关键参数
    return factory;
}
```

**prefetchCount 的选择**：
- = 1：公平分发，适合处理时间不均匀的任务，但吞吐量低
- > 1：批量拉取，吞吐量高，但可能出现"慢消费者囤积任务"

### 批量操作

```
批量发布：合并多个消息到单次网络帧
批量 Confirm：multiple=true 一次性确认多条消息
批量 ACK：消费者处理一批后统一 ACK
```

---

## 7. 常见问题排查

1. **消息丢失**
   - 检查队列 `durable=true`
   - 检查消息 `delivery_mode=2`（Spring Boot 通过 `spring.rabbitmq.template.mandatory=true` + `MessageDeliveryMode.PERSISTENT`）
   - 检查是否开启了 Confirm 模式

2. **消息堆积**
   - 消费者处理速度跟不上 → 增加消费者数量（`concurrency` 参数）
   - 消费者处理异常一直失败 → 检查 `maxLength` + 死信队列兜底

3. **消息重复消费**
   - 消费者 ACK 超时后消息重新入队
   - **根本解决方案**：生产者幂等（每条消息 `correlationId` 唯一）+ 消费者侧去重（数据库唯一键 / Redis 原子操作）

4. **Linux 文件句柄限制**
   - RabbitMQ 打开大量文件（队列、连接），超过系统限制会崩溃
   - 检查 `ulimit -n`，调整 `/etc/security/limits.conf`

5. **性能瓶颈定位**
   - 使用 `rabbitmqctl status` 查看内存、磁盘、连接数
   - 管理界面监控 `Queued Messages` / `Message Rates`

6. **Spring Boot 连接 RabbitMQ 失败**
   - 检查 `spring.rabbitmq.host`、`port`（5672 不是 15672）
   - 检查 Virtual Host 是否存在（`spring.rabbitmq.virtual-host` 默认 `/`）
   - 检查防火墙和认证凭据
