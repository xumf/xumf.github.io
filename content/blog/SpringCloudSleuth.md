---
title : "Spring Cloud Sleuth"
date : "2024-08-01T13:54:50+08:00"
description : "github"
tags : [
  "Spring Cloud Sleuth",
  "Spring Cloud"
]
---
## Spring Cloud Sleuth 核心原理
### 1. 分布式追踪概念
   * Trace：一次完整的请求链路
   * Span：一个工作单元，包含开始和结束时间
   * Annotation：关键事件标记（如 cs、sr、ss、cr）

```Plain Text
cs (Client Sent)：标识客户端发起请求的时间点。
sr (Server Received)：标识服务端接收到请求并开始处理的时间点。
ss (Server Sent)：标识服务端处理完请求并准备返回响应的时间点。
cr (Client Received)：标识客户端接收到服务端响应的时间点。
```
### 2. 核心功能
   * 自动生成和传播 Trace ID 和 Span ID
   * 集成 Zipkin 等追踪系统
   * 支持多种通信协议（HTTP、MQ、RPC等）
   * 提供丰富的上下文信息
### 3. 数据模型
```Plain Text
Trace
├── Span 1
│   ├── Annotation: cs
│   ├── Annotation: sr
│   ├── Annotation: ss
│   └── Annotation: cr
└── Span 2
    ├── Annotation: cs
    └── Annotation: cr
```
## 源码分析
### 1. Tracer 核心类
```java
public interface Tracer {
    Span nextSpan();
    Span nextSpan(Span parent);
    Span newTrace();
    void continueSpan(Span span);
    void close(Span span);
    Span currentSpan();
    // ... other methods ...
}
```
   * 负责创建和管理 Span
   * 提供当前 Span 的访问
   * 支持 Span 的继续和关闭
### 2. TraceContext 传播
```java
public interface Propagator {
    <C> void inject(TraceContext context, C carrier, Setter<C> setter);
    <C> Span.Builder extract(C carrier, Getter<C> getter);
}
```
   * 负责 TraceContext 的注入和提取
   * 支持多种传播方式（如 HTTP Headers）
### 3. Span 创建
```java
public interface Span {
    boolean isNoop();
    Span start();
    Span name(String name);
    Span tag(String key, String value);
    Span event(String value);
    Span error(Throwable throwable);
    void end();
    // ... other methods ...
}
```
   * 表示一个工作单元
   * 支持添加标签和事件
   * 支持错误记录
### 4. 自动配置
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
    
    // ... other beans ...
}
```
   * 自动配置 Tracer 和相关组件
   * 支持通过配置启用/禁用

## 重要知识点
### 1. Sleuth 如何实现分布式追踪？
   * 自动生成传播 Trace ID 和 Span ID
   * 通过 HTTP Headers 或者消息头传播上下文
   * 集成 Zipkin 等追踪系统
### 2. Trace 和 Space 的区别是什么？
   * Trace 表示一次完整的请求链路
   * Span 表示一个工作单元
   * 一个 Trace 包含多个 Span
### 3. 如何自定义 Span 信息？
```java
// ... existing code ...
@Autowired
private Tracer tracer;

public void customSpan() {
    Span span = tracer.nextSpan().name("custom-operation").start();
    try (Tracer.SpanInScope ws = tracer.withSpan(span.start())) {
        // business logic
        span.tag("custom-tag", "value");
    } finally {
        span.end();
    }
}
// ... existing code ...
```
### 4. Sleuth 如何与 Zipkin 集成？
   * 通过 **Spring-cloud-starter-zipkin** 依赖
   * 配置 Zipkin 服务器地址
   * 自动发送追踪数据到 Zipkin
### 5. 如何实现跨服务追踪
   * 通过 HTTP Header 传播 TraceContext
   * 支持 RestTemplate、Feign、WebClient 等
   * 自动处理消息队列的上下文传播
### 6. Sleuth 的性能影响如何？
   * 轻量级实现，性能开销小
   * 支持采样率配置
   * 异步发送追踪数据
### 7. 如何配置采样率？
```yaml
spring:
  sleuth:
    sampler:
      probability: 1.0 # 1.0表示100%采样
```
### 8. Sleuth 支持哪些通信协议？
   * HTTP（RestTemplate、Feign、WebClient）
   * 消息队列（RabbitMQ、Kafka）
   * RPC（gRPC）
   * 数据库访问
### 9. 如何查看 Sleuth 的追踪日志？
   * 日志中会自动添加 **\[appname, traceId,  spanId,  exportable\]**
   * 示例：**\[myapp,2485ec27856c56f4,2485ec27856c56f4,true\]**
### 10. Sleuth 如何处理异步操作？
   * 使用 Tracer.withSpan() 保持上下文
   * 支持 Reactor 上下文传播
   * 示例：

```java
// ... existing code ...
Mono.just("value")
    .doOnNext(value -> {
        Span span = tracer.nextSpan().name("async-operation").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span.start())) {
            // async logic
        } finally {
            span.end();
        }
    })
    .contextWrite(Context.of(Span.class, span));
// ... existing code ...
```
