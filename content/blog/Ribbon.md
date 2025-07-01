---
title: "Ribbon"
date: 2023-04-19T21:09:25+08:00
tags: ["Ribbon", "Spring Cloud"]
---

## 核心概念
主要功能：负载均衡、故障转移、重试机制

特点：

* 客户端负载均衡：负载均衡逻辑在客户端实现，而不是在服务端。
* 集成Spring Cloud：与 Spring Cloud 深度集成，常用于 Feign 和 RestTemplate
* 可扩展性：支持自定义负载均衡策略和规则

## 工作原理
1. 服务发现
   1. Ribbon 从服务注册中心（如 Eureka）获取服务实例列表
   2. 服务实例列表动态更新，确保 Ribbon 能够感知服务实例变化
2. 负载均衡
   1. Ribbon 根据配置的负载均衡策略，从服务实例列表中选择一个实例
   2. 常见的负载均衡策略
      1. 轮询（Round Robin）：依次选择每个实例
      2. 随机（Random）：随机选择一个实例
      3. 加权响应时间（Weighted Response Time）：根据实例的响应时间分配权重
      4. 区域感知（Zone Aware）：优先选择同一区域的实例
3. 故障转移
   1. 如果选择的实例不可用，Ribbon 会自动重试其他实例
   2. 重试次数和超时时间可以通过配置调整
4. 请求分发
   1. Ribbon 将请求分发到选定的服务实例，并处理响应

## 配置
1. 负载均衡策略
   1. 通过配置文件负载均衡策略：

```yaml
service-name:
    ribbon:
        NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule
```
   2. 通过代码配置负载均衡策略：

```java
@RibbonClient(name = "service-name", configuration = MyRibbonConfig.class)
public class MyRibbonConfig {
    @Bean
    public IRule ribbonRule() {
        return new RoundRobinRule(); // 轮询策略
        // return new RandomRule(); // 随机策略
        // return new WeightedResponseTimeRule(); // 加权响应时间策略
        // return new ZoneAvoidanceRule(); // 区域感知策略
    }
}
```
2. 重试机制
   1. 通过配置文件配置重试次数和超时时间

```yaml
  service-name:
    ribbon:
      MaxAutoRetries: 3
      MaxAutoRetriesNextServer: 1
      ConnectTimeout: 1000
      ReadTimeout: 3000
```
   2. 通过代码方式配置重试次数和超时时间

```java
@RibbonClient(name = "service-name", configuration = MyRibbonConfig.class)
public class MyRibbonConfig {
    @Bean
    public IRule ribbonRule() {
        return new RoundRobinRule(); // 轮询策略
    }

    @Bean
    public RetryHandler retryHandler() {
        return new DefaultLoadBalancerRetryHandler(3, 1, true); // 重试配置
    }
}
```
3. 自定义配置
   3. 通过 **@RibbonClient **注解指定自定义配置：

```java
  @RibbonClient(name = "service-name", configuration = MyRibbonConfig.class)
  public class MyRibbonConfig {
      @Bean
      public IRule ribbonRule() {
          return new RandomRule();
      }
  }
```
## 
## 使用场景

1. 与 RestTemplate 集成
   1. 使用 **@LoadBalanced** 注解启用 Ribbon 负载均衡：

```java
  @Bean
  @LoadBalanced
  public RestTemplate restTemplate() {
      return new RestTemplate();
  }
```
   2. 调用服务时，Ribbon 会自动选择服务实例

```java
  String response = restTemplate.getForObject("http://service-name/endpoint", String.class);
```
2. 与 Feign 集成
   1. Feign 默认集成了 Ribbon，无需额外配置
   2. 通过 Feign 客户端调用服务时，Ribbon 会自动处理负载均衡：

```java
  @FeignClient(name = "service-name")
  public interface MyFeignClient {
      @GetMapping("/endpoint")
      String getResponse();
  }
```


## 底层原理
1. 负载均衡器（LoadBalancer）
   1. Ribbon 的核心组件时 **ILoadBalancer**，负责管理服务实例列表和选择实例
   2. 默认实现时** DynamicServerListLoadBalancer **，动态更新服务实例列表

```java
public class DynamicServerListLoadBalancer<T extends Server> extends BaseLoadBalancer {
    private final ServerList<T> serverList; // 服务实例列表
    private final ServerListUpdater serverListUpdater; // 服务实例列表更新器

    @Override
    public void setServersList(List<T> servers) {
        super.setServersList(servers); // 更新服务实例列表
    }

    protected void updateListOfServers() {
        List<T> servers = serverList.getUpdatedListOfServers(); // 获取最新的服务实例列表
        setServersList(servers); // 更新负载均衡器中的服务实例列表
    }

    @Override
    public void start() {
        super.start();
        serverListUpdater.start(new UpdateAction()); // 启动服务实例列表更新任务
    }

    private class UpdateAction implements ServerListUpdater.UpdateAction {
        @Override
        public void doUpdate() {
            updateListOfServers(); // 执行服务实例列表更新
        }
    }
}
```
2. 负载均衡策略（IRule）
   1. **IRule** 接口定义了负载均衡策略，常见实现包括：
      1. **RoundRobinRule**：轮询策略
      2. **RandomRule**：随机策略
      3. **WeightedResponseTimeRule**：加权响应时间策略
      4. **ZoneAvoidanceRule**：区域感知策略
3. 服务实例列表（ServerList）
   1. **ServerList** 接口负责从服务注册中心获取服务实例列表
   2. 默认实现时 **DiscoveryEnabledNIWSServerList**，从 Eureka 获取实例列表

```java
public class DiscoveryEnabledNIWSServerList extends AbstractServerList<DiscoveryEnabledServer> {
    private final EurekaClient eurekaClient; // Eureka 客户端
    private final String serviceId; // 服务名称

    @Override
    public List<DiscoveryEnabledServer> getInitialListOfServers() {
        return obtainServersViaDiscovery(); // 从 Eureka 获取初始服务实例列表
    }

    @Override
    public List<DiscoveryEnabledServer> getUpdatedListOfServers() {
        return obtainServersViaDiscovery(); // 从 Eureka 获取更新后的服务实例列表
    }

    private List<DiscoveryEnabledServer> obtainServersViaDiscovery() {
        List<DiscoveryEnabledServer> serverList = new ArrayList<>();
        if (eurekaClient == null) {
            return serverList;
        }

        // 从 Eureka 获取服务实例信息
        List<InstanceInfo> listOfInstanceInfo = eurekaClient.getInstancesByVipAddress(serviceId, false);
        for (InstanceInfo instanceInfo : listOfInstanceInfo) {
            if (instanceInfo.getStatus() == InstanceInfo.InstanceStatus.UP) {
                // 将 Eureka 的 InstanceInfo 转换为 Ribbon 的 Server
                serverList.add(new DiscoveryEnabledServer(instanceInfo, false, true));
            }
        }
        return serverList;
    }
}
```
4. 故障转移（Retry）
   1. Ribbon 通过 **RetryHandler** 实现故障转移，支持重试和超时处理