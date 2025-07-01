---
title : "Spring Cloud Gateway"
date : "2024-07-18T23:13:45+08:00"
tags : [
  "Spring Cloud Gateway",
  "Spring Cloud",
]
---
## Spring Cloud Gateway 核心原理
1. 架构组成

```Plain Text
Client -> Gateway -> Route Predicate -> Gateway Filter -> Target Service
```
2. 核心概念

* Route：路由，包含、目标 URI、Predicate 结合和 Filter 集合
* Predicate：断言，用于匹配 Http 请求
* Filter：过滤器，用于修改请求和响应
3. 工作流程

```Plain Text
1. 客户端发起请求
2. Gateway Handler Mapping 查找匹配的路由
3. 执行 Gateway Filter 链（pre filters）
4. 代理请求到目标服务
5. 执行 Gateway Filter 链（post filters）
6. 返回响应给客户端
```
4. 关键组件

* RouteLocator：路由定位器
* GatewayFilter：网关过滤器
* RoutePredicateHandlerMapping：路由断言处理器映射
* WebHandler：Web 请求处理器

## 重要知识点
1. Spring Cloud Gateway 和 Zuul 的区别是什么？
   * 架构模型：Gateway 是异步非阻塞，Zuul 1.x 是同步阻塞
   * 性能：Gateway 性能更高
   * 过滤器：Gateway 的过滤器更轻量
   * 协议支持：Gateway 原生支持 HTTP/2
2. Gateway 的核心组件有哪些？
   * Route
   * Predicate
   * Filter
   * RouteLocator
   * GatewayFilterFactory
3. 如何自定义过滤器

```java
// ... existing code ...
@Component
public class CustomFilter implements GatewayFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 前置处理
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            // 后置处理
        }));
    }

    @Override
    public int getOrder() {
        return -1; // 过滤器顺序
    }
}
// ... existing code ...
```
4. Gateway 如何实现负载均衡？
   * 集成 Spring Cloud LoadBalancer
   * 使用 **lb://service-id** 格式的 URI
   * 自动从注册中心获取服务实例
5. 如何实现限流
   * 使用 RequestRateLimiter GatewayFilter
   * 集成 Redis 实现分布式限流
   * 配置示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: limit_route
        uri: http://example.org
        filters:
        - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 10
            redis-rate-limiter.burstCapacity: 20
```
6. Gateway 如何实现熔断？
   * 使用 Hystrix GatewayFilter
   * 配置 fallback 机制
   * 示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: hystrix_route
        uri: lb://backing-service:8088
        filters:
        - name: Hystrix
          args:
            name: fallbackcmd
            fallbackUri: forward:/fallback
```
7. 如何实现动态路由？
   * 实现 RouteDefinitionRepository 接口
   * 从数据库或配置中心加载路由配置
   * 支持实时更新路由规则
8. Gateway 的安全机制有哪些？
   * 集成 Spring Security
   * 支持 OAuth2、JWT 等认证方式
   * 支持 HTTPS
   * 支持 IP 白名单
9. Gateway 的监控如何实现？
   * 集成 Actuator
   * 使用 Micrometer 收集指标
   * 支持 Prometheus 监控
   * 提供 /actuator/gateway 端点
10. Gateway 如何处理跨域问题？
   * 配置全局 CORS
   * 示例：

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins: "*"
            allowedMethods:
            - GET
            - POST
```
