---
title : "Spring Cloud Sleuth"
date : "2024-08-01T13:54:50+08:00"
description : "分布式链路追踪"
tags : [
  "Spring Cloud Sleuth",
  "Spring Cloud"
]
---

> **注意**：Spring Cloud Sleuth 已在 Spring Boot 3.x / Spring Cloud 2022.0.x 中被 **Micrometer Tracing** 取代。新项目直接使用 Micrometer Tracing，旧项目建议迁移。本文基于 Sleuth 2.x 版本。

## 分布式追踪核心概念

### 为什么需要链路追踪？

微服务架构中，一个请求可能需要经过多个服务。当请求变慢或报错时，没有链路追踪很难定位问题出在哪个服务、哪个环节。链路追踪通过 Trace ID 将一次请求跨服务的调用串联起来。

### 核心术语

* **Trace**：一次完整的请求链路，用唯一的 Trace ID 标识
* **Span**：一个工作单元（如一次 RPC 调用、一次数据库查询），包含开始和结束时间
* **Annotation**：关键事件标记，记录请求在不同阶段的时间戳

```
Trace (traceId=abc)
├── Span 1 (id=1, parentId=null) —— 入口服务
│   ├── cs (Client Sent): 时间戳 T1
│   ├── sr (Server Received): 时间戳 T2 —— 下游服务收到
│   ├── ss (Server Sent): 时间戳 T3 —— 下游服务返回
│   └── cr (Client Received): 时间戳 T4
└── Span 2 (id=2, parentId=1) —— 下游服务调数据库
    ├── cs: 时间戳 T5
    └── cr: 时间戳 T6
```

通过这 4 个时间点可以计算出：
- **网络延迟**：sr - cs（请求发送耗时）+ cr - ss（响应传输耗时）
- **服务处理时间**：ss - sr（下游服务实际处理耗时）
- **总耗时**：cr - cs

## Sleuth 核心功能

1. 自动生成并传播 Trace ID 和 Span ID（通过 HTTP Header 或消息头）
2. 集成 Zipkin 等追踪后端
3. 支持多种通信协议（HTTP、消息队列、gRPC）
4. 日志自动关联——日志中自动添加 `[appname, traceId, spanId, exportable]`

```
2024-08-01 10:00:00.123 [http-nio-8080-exec-1] [myapp, 2485ec27856c56f4, 2485ec27856c56f4, true] INFO  - 请求开始
```

通过这个格式，可以按 traceId 筛选出某个请求在所有服务中的完整日志。

## 源码分析

### Tracer 接口

```java
public interface Tracer {
    Span nextSpan();
    Span nextSpan(Span parent);
    Span newTrace();
    void continueSpan(Span span);
    void close(Span span);
    Span currentSpan();
}
```

Tracer 负责 Span 的创建和管理。`newTrace()` 创建全新的 Trace（入口请求），`nextSpan()` 基于当前 Trace 创建子 Span。

### TraceContext 传播

```java
public interface Propagator {
    <C> void inject(TraceContext context, C carrier, Setter<C> setter);
    <C> Span.Builder extract(C carrier, Getter<C> getter);
}
```

`inject` 将 TraceContext 写入 HTTP Header（如 `X-B3-TraceId`），`extract` 从请求头中提取并恢复上下文。这是跨服务传播的核心接口。

### 自动配置

```java
@Configuration
@ConditionalOnProperty(value = "spring.sleuth.enabled", matchIfMissing = true)
public class TraceAutoConfiguration {
    @Bean
    public Tracer tracer() {
        return new DefaultTracer();
    }

    @Bean
    public CurrentTraceContext currentTraceContext() {
        return ThreadLocalCurrentTraceContext.create();
    }

}
```

## 重要知识点

### 1. Sleuth 如何实现跨服务上下文传播？

Sleuth 通过 Brave（Zipkin 的 Java 客户端库）实现上下文传播。HTTP 请求经过 RestTemplate / Feign / WebClient 时，Sleuth 自动注入以下 Header：

- `X-B3-TraceId`：全局唯一的 Trace ID
- `X-B3-SpanId`：当前 Span ID
- `X-B3-ParentSpanId`：父 Span ID
- `X-B3-Sampled`：是否采样
- `X-B3-Flags`：调试标记

下游服务收到请求后，从 Header 中提取 `X-B3-TraceId`，创建子 Span 并将自己作为当前 Trace 的一部分。

### 2. Trace 和 Span 的关系？

- 一个 Trace 由多个 Span 组成，形成**树状结构**
- 根 Span（第一个 Span）没有 parentId
- 子 Span 通过 parentId 引用父 Span
- 所有 Span 共享同一个 traceId

### 3. 如何自定义 Span？

```java
@Autowired
private Tracer tracer;

public void customSpan() {
    Span span = tracer.nextSpan().name("custom-operation").start();
    try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
        // 业务逻辑
        span.tag("custom-key", "custom-value");
        span.annotate("custom-event");
    } finally {
        span.finish();
    }
}
```

注意：`start()` 只调用一次，`finish()` 会在 finally 中确保执行。使用 `withSpanInScope` 将当前 Span 绑定到线程上下文。

### 4. Sleuth 与 Zipkin 集成？

```yaml
spring:
  zipkin:
    base-url: http://localhost:9411
    sender:
      type: web     # 发送方式: web/kafka/rabbitmq
  sleuth:
    sampler:
      probability: 1.0  # 采样率: 0~1，生产环境建议 0.1 以下
```

采样率说明：
- `1.0`：100% 采样，开发和调试时使用
- `0.1`：10% 采样，生产环境常用——追踪数据量大时存储和网络压力不可忽视
- `0.01`：1% 采样，高流量系统的保守选择

### 5. Sleuth 对各种通信方式的支持？

| 通信方式 | 支持情况 | 说明 |
|---------|---------|------|
| RestTemplate | 自动 | 通过 ClientHttpRequestInterceptor |
| Feign | 自动 | 通过 Feign 的 RequestInterceptor |
| WebClient | 自动 | 通过 ExchangeFilterFunction |
| RabbitMQ | 自动 | 通过消息头传播 |
| Kafka | 自动 | 通过消息头传播 |
| gRPC | 依赖扩展 | 需额外配置 |
| 线程池 | 需要手动 | 使用 `Tracer.withSpanInScope` 或 `LazyTraceExecutor` |

### 6. Sleuth 的性能影响？

- Sleuth 本身非常轻量，主要开销在：
  - Trace ID 的生成和传播（极低）
  - 数据发送到 Zipkin（异步，不阻塞请求线程）
  - 采样率高时，Zipkin 存储可能成为瓶颈
- 生产环境建议：`probability: 0.01~0.1` + 使用 Kafka 作为 Reporter（避免 HTTP 发送阻塞）

### 7. Sleuth 如何支持异步操作？

```java
// 方法一：手动绑定上下文
Mono.just("value")
    .doOnNext(value -> {
        Span span = tracer.nextSpan().name("async-operation").start();
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            // 异步逻辑
        } finally {
            span.finish();
        }
    });

// 方法二：使用 Sleuth 提供的 LazyTraceExecutor
@Bean
public Executor asyncExecutor() {
    return new LazyTraceExecutor(tracer, Executors.newFixedThreadPool(10));
}
```

## 排查思路

1. **Trace ID 未在日志中出现**
   - 确认 `spring.sleuth.enabled` 未被设置为 false
   - 确认日志格式正确（Sleuth 通过 `MDC` 注入 traceId）
   - 检查是否使用了自定义 RestTemplate 或 Feign，可能覆盖了默认拦截器

2. **调用链不完整**
   - 确认所有参与链路的服务都配置了 Sleuth
   - 确认服务间的调用方式（HTTP/消息队列）被 Sleuth 支持
   - 检查异步操作中是否丢失了上下文——使用 `Tracer.withSpanInScope` 或 `LazyTraceExecutor`

3. **Zipkin 看不到数据**
   - 检查 `spring.zipkin.base-url` 是否正确
   - 检查网络连通性（从应用程序到 Zipkin Server）
   - 检查采样率设置：`probability: 0` 会导致不发送任何数据

4. **迁移到 Micrometer Tracing 的注意点**
   - 配置从 `spring.sleuth.*` 变为 `management.tracing.*`
   - Trace ID 格式从 64-bit（Sleuth 默认）变为 128-bit（Micrometer 默认）
   - 依赖从 `spring-cloud-starter-sleuth` 变为 `io.micrometer:micrometer-tracing-bridge-brave`
