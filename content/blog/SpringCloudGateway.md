---
title : "Spring Cloud Gateway"
date : "2024-07-18T23:13:45+08:00"
tags : [
  "Spring Cloud Gateway",
  "Spring Cloud",
]
---

## Spring Cloud Gateway 核心原理

### 1. 为什么选择 Gateway 而非 Zuul？

Spring Cloud Gateway 基于 **Spring WebFlux** 和 **Project Reactor**，采用反应式编程模型（非阻塞 I/O）。与 Zuul 1.x（基于 Servlet API，同步阻塞）相比：

| 特性 | Gateway | Zuul 1.x | Zuul 2.x |
|------|---------|----------|----------|
| 底层模型 | Netty + WebFlux | Tomcat + Servlet | Netty |
| I/O 模型 | 异步非阻塞 | 同步阻塞 | 异步非阻塞 |
| 长连接 | 原生支持 | 不支持 | 支持 |
| 性能（吞吐量） | 高 | 低（每个请求占用一个线程） | 中 |
| 学习成本 | 较高（需了解 Reactor） | 低 | 较高 |

Gateway 的异步模型意味着：一个请求从进入到返回不占用独立线程，处理过程中线程可以服务其他请求，因此在高并发场景下线程资源占用远少于 Zuul 1.x。

### 2. 架构组成

```plain Text
Client -> Gateway -> Route Predicate -> Gateway Filter -> Target Service
```

### 3. 核心概念

- **Route**：路由，包含目标 URI、Predicate 集合和 Filter 集合
- **Predicate**：断言，用于匹配 HTTP 请求（满足条件才走该路由）
- **Filter**：过滤器，用于修改请求和响应（分为 GatewayFilter 和 GlobalFilter）

### 4. 工作流程

```plain Text
1. 客户端发起请求
2. Gateway Handler Mapping 查找匹配的路由（遍历所有 Route 的 Predicate 集合）
3. 执行 Gateway Filter 链（pre filters）
4. 代理请求到目标服务
5. 执行 Gateway Filter 链（post filters）
6. 返回响应给客户端
```

### 5. 关键组件

- **RouteLocator**：路由定位器，定义路由规则
- **GatewayFilter**：网关过滤器（按路由配置）
- **GlobalFilter**：全局过滤器（所有路由生效）
- **RoutePredicateHandlerMapping**：路由断言处理器映射
- **WebHandler**：Web 请求处理器（底层由 WebFlux 的 DispatcherHandler 执行）

## 重要知识点

### 1. Gateway 和 Zuul 的区别？

见上方对比表。核心差异在于**线程模型**：Gateway 用少量线程处理大量请求（事件驱动），Zuul 1.x 每个请求对应一个线程（同步阻塞）。

### 2. Gateway 的核心组件？

Route、Predicate、Filter（GatewayFilter + GlobalFilter）、RouteLocator、GatewayFilterFactory

### 3. 如何自定义过滤器？

自定义 GatewayFilter 实现 GatewayFilter + Ordered 接口：

```java
@Component
public class CustomFilter implements GatewayFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 前置处理
        String requestPath = exchange.getRequest().getURI().getPath();
        exchange.getAttributes().put("startTime", System.currentTimeMillis());

        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            // 后置处理
            Long startTime = exchange.getAttribute("startTime");
            if (startTime != null) {
                long cost = System.currentTimeMillis() - startTime;
                log.info("{} 耗时 {}ms", requestPath, cost);
            }
        }));
    }

    @Override
    public int getOrder() {
        return -1; // 越小越先执行
    }
}
```

注册方式：在 Route 定义中通过 `.filters(f -> f.filter(new CustomFilter()))` 注入。

### 4. Gateway 如何实现负载均衡？

- 集成 Spring Cloud LoadBalancer
- 使用 **lb://service-id** 格式的 URI（例如 `lb://user-service`）
- Gateway 自动从注册中心（Nacos / Eureka）获取服务实例列表
- 默认负载均衡算法为轮询（支持自定义 ReactiveLoadBalancer）

### 5. 如何实现限流？

使用 RequestRateLimiter GatewayFilter，基于令牌桶算法：

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
            redis-rate-limiter.replenishRate: 10  # 每秒填充的令牌数
            redis-rate-limiter.burstCapacity: 20   # 令牌桶容量
```

注意：GateWay 的限流需要集成 Redis（基于 Redis 的 lua 脚本实现令牌桶），因此必须引入 `spring-boot-starter-data-redis-reactive`。

### 6. Gateway 如何实现熔断？

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

注意：Hystrix 已进入维护状态，新项目推荐使用 **Spring Cloud CircuitBreaker**（支持 Resilience4J）替代 Hystrix：

```yaml
filters:
- name: CircuitBreaker
  args:
    name: myCircuitBreaker
    fallbackUri: forward:/fallback
```

### 7. 如何实现动态路由？

实现 `RouteDefinitionRepository` 接口，从数据库或配置中心加载路由：

```java
@Component
public class DynamicRouteService implements ApplicationEventPublisherAware {
    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;

    public void addRoute(RouteDefinition definition) {
        routeDefinitionWriter.save(Mono.just(definition)).subscribe();
        publisher.publishEvent(new RefreshRoutesEvent(this)); // 刷新路由
    }
}
```

更新的关键：保存路由定义后必须发布 `RefreshRoutesEvent`，否则新路由不会生效。

### 8. Gateway 的安全机制？

- 集成 Spring Security + OAuth2 / JWT
- 支持 HTTPS
- 支持 IP 白名单（通过自定义 GlobalFilter 实现）
- 常见安全风险：Gateway 默认不校验重复路由、不限制请求体大小——生产环境需额外配置

### 9. Gateway 的监控？

- 集成 Actuator，`management.endpoints.web.exposure.include=gateway`
- `/actuator/gateway/routes`：查看已加载的路由
- `/actuator/gateway/globalfilters`：查看已注册的全局过滤器
- 结合 Micrometer + Prometheus + Grafana 采集请求延迟和错误率

### 10. Gateway 如何处理跨域？

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
            - PUT
            - DELETE
            allowedHeaders: "*"
```

注意：Gateway 的位置特殊——跨域配置在网关层处理即可，后端服务无需重复配置 CORS。

## 排查思路

1. **路由不匹配**
   - 检查 Predicate 顺序（多个 Predicate 为 AND 关系）
   - 查看日志：`RoutePredicateHandlerMapping` 会打印 "Mapping found" 或 "No route found"
2. **转发失败**
   - 确认目标服务健康（直接访问目标服务接口验证）
   - 检查负载均衡 URI 格式是否以 lb:// 开头
   - 查看 Gateway 日志：`ReadTimeoutException` 表示后端响应过慢
3. **过滤器不生效**
   - GlobalFilter 默认所有路由生效，GatewayFilter 须在路由配置中明确指定
   - 检查 Ordered 排序（低 order 值的过滤器先执行前置后执行后置）
4. **WebFlux 与旧版 Servlet 冲突**
   - 项目中不能同时存在 `spring-boot-starter-web` 和 `spring-boot-starter-webflux`，否则 Gateway 无法正常运行（Spring Boot 默认优先使用 Servlet 容器）。排查是否有其他模块引入了 web 依赖。
