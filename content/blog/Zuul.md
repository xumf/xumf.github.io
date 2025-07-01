---
title : "Zuul"
date : "2025-05-07T23:04:30+08:00"
tags : [
  "Spring Cloud",
  "Zuul"
]

---

## Zuul 的核心概念
### （1）Zuul 是什么？
* Zuul 是 Netflix 开源的 API 网关，用于微服务架构中的路由、过滤和负载均衡
* 主要功能：
   * 路由：将客户端请求转发到后端服务
   * 过滤：在请求和响应的生命周期中执行自定义逻辑
   * 负载均衡：与 Ribbon 集成，实现客户端负载均衡

### （2）Zuul 的工作流程
1. 路由：根据配置请求转发到对应的微服务
2. 过滤：在请求到达目标服务之前或之后执行过滤逻辑
3. 负载均衡：通过 Ribbon 集成，实现客户端负载均衡

## Zuul 的配置
### （1）依赖配置
* 在 pom.xml 添加 Zuul 依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
```
### （2）启用 Zuul
* 在启动类中添加 **@EnableZuulProxy** 注解：

```java
@SpringBootApplication
@EnableZuulProxy
public class ZuulApplication {
    public static void main(String[] args) {
        SpringAppliction.run(ZuulApplication.class, args);
    }
}
```
### （3）路由配置
* 在 application.yml 中配置路由规则：

```java
zuul:
    routes:
        user-service: # 路径名称
            path: /user/** # 路径配置
            serviceId: user-service # 目标服务名称
        order-service:
            path: /order/**
            serviceId: order-service
```
* 解释：
   * user-service 和 order-service 是路由的名称，可以自定义
   * path 是匹配的 URL 路径
   * serviceId 是 Eureka 中注册的服务名称
* 自定义路由
   * 如果不想使用服务名称作为路由前缀，可以自定义路由：

```yaml
zuul:
    routes:
        custom-user-service: # 路由策略
            path: /api/user/** # 匹配路径
            url: http://localhost:8081 # 目标服务地址
```
## 负载均衡配置
### （1）集成 Ribbon
* Zuul 默认集成了 Ribbon，可以通过服务名称实现负载均衡
* 在 **application.yml** 中配置 Ribbon：

```yaml
zuul:
    routes:
        custom-user-service:
            path: /api/user/**
            serviceId: custom-user-service # 使用服务名称
custom-user-service:
    ribbon:
        listOfServers: http://localhost:8081,http://localhost:8082 # 目标服务实例列表
        NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule # 随机策略
```
## 实例：完整的 Zuul 配置
### （1）application.yml
```yaml
user-service:
    ribbon:
        listOfServers: http://localhost:8081,http://localhost:8082 # 目标服务实例列表
```
### （2）启动类
```java
@SpringBootApplication
@EnableZuulProxy
public class ZuulApplication {
    public static void main(String[] args) {
        SpringAppliction.run(ZuulApplication.class, args);
    }
}
```
## Zuul 与 Spring Cloud Gateway对比
|对比项|Spring Cloud Gateway|Spring Cloud Zuul 1.x|
| ----- | ----- | ----- |
|架构模型|异步非阻塞 (基于 WebFlux 和 Reactor)|同步阻塞 (基于 Servlet)|
|过滤器链|轻量高效|相对较重|
|路由匹配|高效的路由匹配算法|相对较慢|
|内存消耗|较低 (基于 Netty)|较高 (基于 Servlet 容器)|
|编程模型|响应式编程|传统同步模型|
|负载均衡|集成 Spring Cloud LoadBalancer|使用 Ribbon|
|线程切换|较少|较多|
|连接管理|使用 Netty 连接池|相对简单|
|协议支持|原生支持 HTTP/2|主要支持 HTTP/1.1|
|上下文切换|较少|较多|
|性能|更高|相对较低|

