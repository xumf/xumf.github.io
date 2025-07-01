---
title : "Zipkin"
date : "2025-03-03T22:19:24+08:00"
description : "Zipkin 原理"
tags : [
  "Zipkin",
  "Spring Cloud"
]

---
## Zipkin 核心原理
### 1. 架构组成
```Plain Text
+-----------+     +-----------+     +-----------+
| Collector | <-- | Reporter | <-- | Instrument|
+-----------+     +-----------+     +-----------+
      |                  |
      v                  v
+-----------+     +-----------+
| Storage   |     | UI        |
+-----------+     +-----------+
```
### 2. 核心概念
   * Trace：一次完整的请求链路
   * span：一个工作单元
   * Annotation：关键事件标记
   * BinaryAnnotation：附加的键值信息

```Plain Text
1. cs (Client Sent): 客户端发送请求的时间。
2. sr (Server Received): 服务器接收到请求的时间。
3. ss (Server Sent): 服务器发送响应的时间。
4. cr (Client Received): 客户端接收到响应的时间。
5. ca (Client Address): 客户端的地址信息。
6. sa (Server Address): 服务器的地址信息。
7. error (Error): 表示请求过程中是否发生了错误。
8. mvc.controller (MVC Controller): 与 Spring MVC 控制器相关的注解。
9. mvc.method (MVC Method): 与 Spring MVC 方法相关的注解。
10. http.method (HTTP Method): HTTP 请求的方法（如 GET, POST 等）。
11. http.path (HTTP Path): HTTP 请求的路径。
12. http.status_code (HTTP Status Code): HTTP 响应的状态码。
13. http.url (HTTP URL): HTTP 请求的完整 URL。
14. sql.query (SQL Query): 执行的 SQL 查询语句。
15. sql.statement (SQL Statement): 执行的 SQL 语句。
16. rabbitmq.message (RabbitMQ Message): 与 RabbitMQ 消息相关的注解。
17. redis.command (Redis Command): 执行的 Redis 命令。
```
### 3. 数据流
```Plain Text
1. 应用通过 Reporter 发送追踪数据
2. Collector 接收并验证数据
3. 数据存储到 Storage
4. UI 从 Storage 查询并展示数据
```
## 源码分析
### 1. Span 数据结构
```java
public final class Span implements Serializable {
    public static Builder newBuilder() {
        return new Builder();
    }
    
    public static final class Builder {
        private String traceId;
        private String parentId;
        private String id;
        private String name;
        private long timestamp;
        private long duration;
        private List<Annotation> annotations;
        private Map<String, String> tags;
        // ... other fields and methods ...
    }
}
```
   * 表示一个追踪单元
   * 包含 traceId、parentId、id 等关键信息
   * 支持添加注解和标签
### 2. Reporter 接口
```java
public interface Reporter<S> {
    void report(S span);
    
    default void close() throws IOException {}
}
```
   * 负责发送追踪数据
   * 支持异步报告
   * 内置多种实现（如 Kafka、RabbitMQ、HTTP）
### 3. Collector 组件
```java
public abstract class Collector {
    public abstract void accept(List<Span> spans);
    
    protected void storeSpans(List<Span> spans) {
        // 存储逻辑
    }
}
```
   * 负责接收和验证追踪数据
   * 调用存储组件持久化数据
### 4. Storage 组件
```java
public interface StorageComponent {
    SpanConsumer spanConsumer();
    
    SpanStore spanStore();
    
    AutocompleteTags autocompleteTags();
}
```
### 5. UI 组件
```java
@RestController
public class ZipkinUi {
    @GetMapping("/")
    public String index() {
        return "index";
    }
    
    @GetMapping("/traces/{id}")
    public String traceDetail(@PathVariable String id) {
        // 查询并返回追踪详情
    }
}
```
   * 提供 Web 界面展示追踪数据
   * 支持查询和可视化

## 重要知识点
### 1. Zipkin 的工作原理是什么？
   * 应用通过 Reporter 发送追踪数据
   * Collector 接收并存储数据
   * UI 提供查询和可视化界面
### 2. Zipkin 支持哪些存储后端？
   * 内存（默认）
   * MySQL
   * ElasticSearch
   * Cassandra
### 3. 如何集成 Zipkin 和 Spring Cloud Sleuth
```yaml
spring:
  zipkin:
    base-url: http://localhost:9411
    sender:
      type: web # 使用HTTP发送
  sleuth:
    sampler:
      probability: 1.0 # 采样率
```
### 4. ZipKin 的数据模型怎么的？
   * Trace：一次性完整的请求链路
   * Span：一个工作单元
   * Annotation：关键事件标记
   * BinaryAnnotation：附加的键值对信息
### 5. 如何自定义 Zipkin 的 Reporter？
```java
// ... existing code ...
@Bean
Reporter<Span> myReporter() {
    return new AsyncReporter<>(URLConnectionSender.create("http://localhost:9411/api/v2/spans"));
}
// ... existing code ...
```
### 6. Zipkin 的新优化策略有哪些？
   * 使用异步 Reporter
   * 配置合适的采样率
   * 选择合适的存储后端
   * 批量发送追踪数据
### 7. 如何实现 Zipkin 的高可用
   * 部署多个 Collector 实例
   * 使用负载均衡
   * 配置持久化存储
   * 设置合理的存储清理策略
### 8.  Zipkin 的 UI 有哪一些？
   * 查看追踪列表
   * 查看单个追踪详情
   * 依赖关系图
   * 服务调用统计
### 9. 如何处理 Zipkin 的数据存储问题？
   * 定期清理旧数据
   * 使用 TTL（Time To Live）
   * 配置合适的存储容量
   * 监控存储性能
### 10. Zipkin 如何支持多语言？
   * 提供统一的 API 接口
   * 支持多种 Reporter 实现
   * 提供多种语言的客户端库
   * 使用通用的数据格式（如 JSON）