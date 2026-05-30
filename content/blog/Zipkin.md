---
title : "Zipkin"
date : "2025-03-03T22:19:24+08:00"
description : "Zipkin 原理"
tags : [
  "Zipkin",
  "Spring Cloud"
]

---

Zipkin 是 Twitter 开源的分布式链路追踪系统，用于收集和查看微服务架构中的请求链路数据。它与 Spring Cloud Sleuth（或 Micrometer Tracing）配合使用——Sleuth 负责生成和传播 Trace 数据，Zipkin 负责存储和展示。

## 架构组成

```plain text
+-----------+     +-----------+     +-----------+
| Collector | <-- | Reporter | <-- | Instrument|
+-----------+     +-----------+     +-----------+
      |                  |
      v                  v
+-----------+     +-----------+
| Storage   |     | UI        |
+-----------+     +-----------+
```

- **Instrument**：应用中嵌入的追踪客户端（Brave / Sleuth），生成 Span 数据
- **Reporter**：将 Span 数据异步发送到 Zipkin Server。支持 HTTP、Kafka、RabbitMQ
- **Collector**：接收、验证、存储 Span 数据
- **Storage**：持久化存储（内存/MySQL/ElasticSearch/Cassandra）
- **UI**：Web 界面查询和可视化链路

## 数据模型

### Span 数据结构

```java
public final class Span implements Serializable {
    public static final class Builder {
        private String traceId;       // 全局唯一链路 ID
        private String parentId;      // 父 Span ID（根 Span 为 null）
        private String id;            // 当前 Span ID
        private String name;          // Span 名称（如 "get /user/{id}"）
        private long timestamp;       // 开始时间戳（微秒）
        private long duration;        // 持续时间（微秒）
        private List<Annotation> annotations;  // 关键事件标记
        private Map<String, String> tags;       // 自定义标签
        private Boolean shared;       // 是否被服务端和客户端共享
    }
}
```

### 关键 Annotation

| 标记 | 含义 | 用途 |
|------|------|------|
| cs (Client Sent) | 客户端发送请求 | 计算请求开始时间 |
| cr (Client Received) | 客户端收到响应 | 计算总耗时 = cr - cs |
| sr (Server Received) | 服务端收到请求 | 计算网络延迟 = sr - cs |
| ss (Server Sent) | 服务端发送响应 | 计算服务处理时间 = ss - sr |
| error | 请求出错 | 标记异常链路 |

### 数据流

```
1. 应用通过 Reporter 发送 Span 数据
2. Collector 接收并校验数据完整性
3. 数据存储到 Storage（可选择 ES/Cassandra/MySQL）
4. UI 从 Storage 查询并展示
```

## Reporter 发送方式对比

| 方式 | 优点 | 缺点 | 场景 |
|------|------|------|------|
| HTTP（Web） | 部署简单，无需额外组件 | 请求阻塞风险，吞吐量低 | 开发和低流量 |
| Kafka | 高吞吐，削峰填谷 | 需要维护 Kafka 集群 | 生产环境高流量 |
| RabbitMQ | 类似 Kafka，与 Spring 生态更适配 | 需要维护 RabbitMQ | 已有 RabbitMQ 的基础设施 |
| Scribe | 日志收集 | 维护成本高 | 历史遗留系统 |

使用 Kafka 推荐配置：

```yaml
spring:
  zipkin:
    sender:
      type: kafka
  kafka:
    bootstrap-servers: localhost:9092
```

Kafka 的优势：即使 Zipkin Server 短暂不可用，数据也不会丢失（Kafka 持久化缓冲）。

## 存储后端选择

| 存储 | 优势 | 劣势 | 推荐场景 |
|------|------|------|---------|
| 内存 | 零配置，快速体验 | 重启后数据丢失，容量有限 | 开发测试 |
| MySQL | 运维成熟 | 查询性能差，数据量大时慢 | 低流量/小团队 |
| ElasticSearch | 查询快，适合全文搜索 | 运维复杂，资源消耗高 | 生产环境（推荐） |
| Cassandra | 写入性能极高，天然支持水平扩展 | 查询功能有限，运维复杂 | 超大规模集群 |

生产环境选择建议：**ElasticSearch** 是最常用的选择，兼顾查询灵活性和写入性能。

## 数据采样策略

### 采样重要性

全量采样（`probability: 1.0`）在高并发下会产生大量数据——1000 QPS 的服务每天产生近亿条 Span。采样是必须的。

### 采样方式

1. **固定比率采样**：默认，按百分比采样

```yaml
spring:
  sleuth:
    sampler:
      probability: 0.01  # 采样 1%
```

2. **限速采样**：保证每秒最多采样条数

```yaml
spring:
  sleuth:
    sampler:
      rate: 10  # 每秒最多采样 10 条
```

3. **自定义采样规则**：对某些路径全量采样（如支付、订单相关接口），其他路径降低采样率。需实现 `Sampler` 接口。

## 集成示例

```yaml
spring:
  zipkin:
    base-url: http://zipkin-server:9411
    sender:
      type: web
    enabled: true
  sleuth:
    sampler:
      probability: 0.1
```

```bash
# 启动 Zipkin Server（Docker）
docker run -d -p 9411:9411 openzipkin/zipkin:latest
```

## 排查思路

1. **Zipkin UI 看不到数据**
   - 检查应用能否访问 Zipkin Server：telnet zipkin-server 9411
   - 检查采样率是否为 0
   - 检查应用日志中是否出现 `ZipkinReporter` 相关的异常

2. **链路不完整**
   - 确认所有参与链路的服务都配置了 Zipkin
   - 检查异步调用是否丢失 TraceContext——异步线程池中需要手动传播上下文
   - 确认 Http Header 未被自定义过滤器清除（`X-B3-TraceId` 等）

3. **Zipkin 存储满了**
   - ES：设置 ILM（Index Lifecycle Management）自动删除过期索引
   - Cassandra：配置 TTL（Time To Live）
   - MySQL：定期清理，或设置较短的数据保留期

4. **性能问题**
   - Zipkin Collector 成为瓶颈：增加 Collector 实例数
   - ES 写入过慢：降低采样率，或使用 Kafka 缓冲
   - 使用 `sender.type: kafka` 将数据发送链路上的压力从应用线程转移到 Kafka

## 高可用部署

```
多个 Zipkin Server 实例
    ↑  (Nginx 负载均衡)
Kafka 集群（缓冲区）
    ↑
应用实例（Reporter 写入 Kafka）
```

- 部署多个 Collector 实例，使用 Nginx/ALB 负载均衡
- 使用 Kafka 作为 Reporter 和 Collector 之间的缓冲层
- 存储后端 ES 集群健康状态决定数据可靠性——使用 3 节点 + 副本策略
- ES 使用 ILM 按天生成索引，设置保留天数自动清理
