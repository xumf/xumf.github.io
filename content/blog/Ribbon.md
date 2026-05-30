---
title: "Ribbon"
date: 2023-04-19T21:09:25+08:00
tags: ["Ribbon", "Spring Cloud"]
---

> **注意**：Ribbon 已进入**维护状态**（Netflix 不再积极开发）。从 Spring Cloud 2020.0.x（Ilford）开始，官方推荐使用 **Spring Cloud LoadBalancer** 作为替代。新项目直接使用 LoadBalancer，旧项目建议逐步迁移。本文内容基于 Ribbon 2.x 版本。

## 核心概念

Ribbon 是 Netflix 开源的客户端负载均衡器，核心功能：

- **负载均衡**：从多个服务实例中选择一个处理请求
- **故障转移**：选择的实例不可用时自动重试其他实例
- **重试机制**：支持配置重试次数和超时

关键设计点：Ribbon 是**客户端负载均衡**，与 Nginx 等**服务端负载均衡**不同：

| 特性 | 客户端负载均衡（Ribbon） | 服务端负载均衡（Nginx） |
|------|------------------------|----------------------|
| 位置 | 在应用内部运行 | 独立部署 |
| 实例感知 | 从注册中心获取列表 | 配置 upstream |
| 故障转移 | 自动重试其他实例 | 需在 upstream 配置 |
| 部署复杂度 | 无需额外部署 | 需要独立部署和运维 |

## 工作原理

1. **服务发现**：Ribbon 从注册中心（如 Eureka）获取服务实例列表，并动态更新
2. **负载均衡**：根据配置的策略从列表中选择一个实例
3. **故障转移**：不可用时自动重试其他实例
4. **请求分发**：将请求分发到选定实例，处理响应

### 完整请求流程

```
Client Request
    → RestTemplate/Feign (URL: http://service-name/endpoint)
    → RibbonLoadBalancerClient.choose("service-name")
        → DynamicServerListLoadBalancer.getServerList()
            → EurekaClient.getInstancesByVipAddress()
        → IRule.choose(serverList)
            → 选择策略: 轮询/随机/加权...
    → 发送 HTTP 请求到选定实例
    → 失败时: RetryHandler 重试其他实例
    → 返回响应
```

## 负载均衡策略详解

Ribbon 提供 7 种内置策略：

| 策略类 | 说明 | 适用场景 |
|-------|------|---------|
| RoundRobinRule | 轮询 | 默认策略，各实例配置均衡 |
| RandomRule | 随机 | 无状态服务 |
| WeightedResponseTimeRule | 加权响应时间 | 实例性能不均 |
| ZoneAvoidanceRule | 区域感知 | 多数据中心部署 |
| BestAvailableRule | 最小并发 | 长连接场景 |
| RetryRule | 带重试的轮询 | 需要自动重试 |
| AvailabilityFilteringRule | 可用性过滤 | 剔除熔断实例 |

## 配置方式

### 全局配置

```yaml
ribbon:
  ConnectTimeout: 1000
  ReadTimeout: 3000
  MaxAutoRetries: 0          # 同一实例重试次数
  MaxAutoRetriesNextServer: 1 # 切换实例重试次数
  OkToRetryOnAllOperations: false  # 是否所有操作都重试
```

### 服务级配置

```yaml
service-name:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule
    MaxAutoRetries: 3
    MaxAutoRetriesNextServer: 1
    ConnectTimeout: 1000
    ReadTimeout: 3000
```

### 代码配置

```java
@RibbonClient(name = "service-name", configuration = MyRibbonConfig.class)
public class MyRibbonConfig {
    @Bean
    public IRule ribbonRule() {
        return new RandomRule();
    }

    @Bean
    public RetryHandler retryHandler() {
        // 参数: maxRetriesOnSameServer, maxRetriesOnNextServer, okToRetryOnAllOperations
        return new DefaultLoadBalancerRetryHandler(3, 1, true);
    }
}
```

**注意**：使用 `@RibbonClient` 时，配置类不能被 `@ComponentScan` 扫描到（否则成为全局配置），建议放在启动类所在包之外。

## 使用场景

### 与 RestTemplate 集成

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// 调用时直接用服务名，Ribbon 自动解析为实例 IP:PORT
String response = restTemplate.getForObject("http://service-name/endpoint", String.class);
```

### 与 Feign 集成

Feign 默认集成 Ribbon，无需额外配置：

```java
@FeignClient(name = "service-name")
public interface MyFeignClient {
    @GetMapping("/endpoint")
    String getResponse();
}
```

## 底层原理

### ILoadBalancer 接口

Ribbon 的核心接口是 `ILoadBalancer`，负责管理服务实例列表：

```java
public interface ILoadBalancer {
    void addServers(List<Server> newServers);
    Server chooseServer(Object key);     // 选择一个实例
    void markServerDown(Server server);
    List<Server> getServerList(boolean availableOnly);
    void markServerDown(Server server);
}
```

默认实现 `DynamicServerListLoadBalancer` 通过 `ServerListUpdater` 定时从注册中心拉取最新实例：

```java
public class DynamicServerListLoadBalancer<T extends Server> extends BaseLoadBalancer {
    private final ServerList<T> serverList;
    private final ServerListUpdater serverListUpdater;

    protected void updateListOfServers() {
        List<T> servers = serverList.getUpdatedListOfServers();
        setServersList(servers);
    }
}
```

### IRule 负载均衡策略接口

```java
public interface IRule {
    Server choose(Object key);
    void setLoadBalancer(ILoadBalancer lb);
    ILoadBalancer getLoadBalancer();
}
```

### 从 Eureka 获取实例

```java
public class DiscoveryEnabledNIWSServerList extends AbstractServerList<DiscoveryEnabledServer> {
    private final EurekaClient eurekaClient;
    private final String serviceId;

    private List<DiscoveryEnabledServer> obtainServersViaDiscovery() {
        List<InstanceInfo> listOfInstanceInfo = eurekaClient.getInstancesByVipAddress(serviceId, false);
        for (InstanceInfo instanceInfo : listOfInstanceInfo) {
            if (instanceInfo.getStatus() == InstanceInfo.InstanceStatus.UP) {
                serverList.add(new DiscoveryEnabledServer(instanceInfo, false, true));
            }
        }
        return serverList;
    }
}
```

## 迁移到 Spring Cloud LoadBalancer

从 Spring Cloud 2020.0.x 开始，Ribbon 被 LoadBalancer 取代。迁移要点：

1. **移除 Ribbon 依赖**：删除 `spring-cloud-starter-netflix-ribbon`
2. **添加 LoadBalancer 依赖**：`spring-cloud-starter-loadbalancer`
3. **接口变化**：
   - `ILoadBalancer` → `ReactiveLoadBalancer` / `BlockingLoadBalancerClient`
   - `IRule` → `ReactiveLoadBalancer` 实现（如 `RoundRobinLoadBalancer`）
   - `@RibbonClient` → 无直接等价，通过 `LoadBalancerClient` 配置
4. **配置变化**：`service-name.ribbon.*` → `spring.cloud.loadbalancer.*`

```yaml
# LoadBalancer 配置示例
spring:
  cloud:
    loadbalancer:
      retry:
        enabled: true
      health-check:
        interval: 30s
      configurations: health-check # 启用健康检查
```

## 排查思路

1. **负载均衡不生效（总是访问同一个实例）**
   - 确认 `@LoadBalanced` 注解已在 RestTemplate Bean 上
   - 检查服务名解析：`http://service-name/` 中的 service-name 是否与注册中心的服务名一致
   - Ribbon 默认使用 ZoneAvoidanceRule，如果所有实例在同一 zone，行为类似轮询

2. **实例列表未更新**
   - 检查 `ServerListRefreshInterval`（默认 30 秒）
   - 检查 Eureka Client 的 `eureka.client.registryFetchIntervalSeconds`（默认 30 秒）

3. **重试不生效**
   - Ribbon 本身不触发重试——需要配合 Spring Retry（需引入 `spring-retry` 依赖）
   - 只有 `@LoadBalanced` 的 RestTemplate 或 Feign 客户端才会触发 Ribbon 重试
   - 确保 `MaxAutoRetries` 和 `MaxAutoRetriesNextServer` 配置了正值

4. **引入 LoadBalancer 后 Ribbon 失效**
   - Spring Cloud 2020.0.x 默认禁用了 Ribbon，如果仍想使用需要设置 `spring.cloud.loadbalancer.ribbon.enabled=true`（但已不推荐）
