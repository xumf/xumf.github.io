---
title: "Spring Cloud Bus"
date: 2024-01-05T13:42:22+08:00
tags: ["Spring Cloud Bus", "Spring Cloud"]
---

## Spring Cloud Bus 的核心概念

### （1）什么是 Spring Cloud Bus？
* Spring Cloud Bus 是一个分布式事件总线，用于在微服务之间传播状态变化（如配置更新）
* 它基于消息中间件（RabbitMQ、Kafka）实现，解决 Config Server 逐台通知的痛点——没有 Bus 时，配置变更后需要逐一重启或手动调用每个客户端端点的 `refresh`
* 与 Spring Cloud Config 配合使用是主流场景，但 Bus 本身不依赖 Config——任何需要广播的事件都可以通过 Bus 传播

### （2）Spring Cloud Bus 的组成
* **消息中间件**：RabbitMQ 或 Kafka，负责传递事件
* **事件生产者**：Config Server 或其他微服务，发送事件
* **事件消费者**：Config Client 或其他微服务，接收事件并执行相应操作

### （3）Spring Cloud Bus 的特点
* 全局配置刷新：通过 `/actuator/bus-refresh` 端点触发全局配置刷新
* 服务状态同步：通过事件传播实现服务状态同步
* 多消息中间件支持：支持 RabbitMQ 和 Kafka
* **定点通知**：支持指定某个或某几个服务实例刷新（`/actuator/bus-refresh/{destination}`），避免全量广播

## Spring Cloud Bus 的工作原理

### （1）事件传播流程
1. 事件生产者（如 Config Server）发送事件到消息中间件
2. 消息中间件将事件广播给所有订阅该队列的消费者
3. 事件消费者接收到事件并执行相应操作

关键：每个 Bus 客户端启动时会在消息中间件中创建自己的唯一队列（基于 `spring.cloud.bus.id`），Bus 通过这个队列实现定点通知——`/bus-refresh/user-service:**` 只会发送给 service ID 为 user-service 的实例。

### （2）全局配置刷新流程
1. 管理员调用 POST `/actuator/bus-refresh`（向任意一个 Bus 客户端请求即可——它会通过消息中间件广播给所有实例）
2. 该客户端发布 `RefreshRemoteApplicationEvent`
3. 消息中间件广播该事件
4. 所有 Config Client 监听到事件，调用 `ContextRefresher.refresh()` → `RefreshScope.refreshAll()`
5. 标记了 `@RefreshScope` 的 Bean 被销毁并重新创建，加载新配置

## Spring Cloud Bus 的配置

### （1）RabbitMQ 配置

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
```

### （2）Kafka 配置

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: my-group
```

### （3）添加依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId> <!-- RabbitMQ -->
    <!-- 或 -->
    <!-- <artifactId>spring-cloud-starter-bus-kafka</artifactId> --> <!-- Kafka -->
</dependency>
```

### （4）暴露 Actuator 端点

```yaml
management:
  endpoints:
    web:
      exposure:
        include: bus-refresh
```

## Spring Cloud Bus 的使用场景

### （1）全局配置刷新

```bash
# 全量刷新
curl -X POST http://localhost:8888/actuator/bus-refresh

# 定点刷新（只刷新 user-service）
curl -X POST http://localhost:8888/actuator/bus-refresh/user-service:**
```

### （2）自定义事件

```java
// 发送自定义事件
@Autowired
private ApplicationEventPublisher publisher;

public void sendCustomEvent() {
    publisher.publishEvent(new MyCustomEvent(this, "custom-data"));
}

// 接收自定义事件
@EventListener
public void handleCustomEvent(MyCustomEvent event) {
    // 处理事件
}
```

注意：自定义事件必须继承 `RemoteApplicationEvent` 才能通过 Bus 传播到其他服务节点，否则只会在本地发布。

### （3）指定服务刷新的场景

当集群中不同服务配置不同时，全量刷新可能造成混乱。定点刷新适用于：
- 灰度配置发布：只刷新新版本的服务实例
- 分批部署：一组一组刷新，观察无问题后再继续

## Spring Cloud Bus 源码概览

### （1）BusRefreshEndpoint

```java
@Endpoint(id = "bus-refresh")
public class BusRefreshEndpoint {

    private final ApplicationEventPublisher publisher;

    public BusRefreshEndpoint(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    @WriteOperation
    public void refresh() {
        publisher.publishEvent(new RefreshRemoteApplicationEvent(this, null, null));
    }
}
```

### （2）RefreshRemoteApplicationEvent

```java
public class RefreshRemoteApplicationEvent extends RemoteApplicationEvent {
    public RefreshRemoteApplicationEvent(Object source, String originService, String destinationService) {
        super(source, originService, destinationService);
    }
}
```

`destinationService` 参数用于定点通知：`null` 表示广播给所有服务，指定值（如 `user-service:**`）则只发送给匹配的实例。

### （3）RefreshListener

```java
public class RefreshListener implements ApplicationListener<RefreshRemoteApplicationEvent> {

    private final RefreshScope refreshScope;

    public RefreshListener(RefreshScope refreshScope) {
        this.refreshScope = refreshScope;
    }

    @Override
    public void onApplicationEvent(RefreshRemoteApplicationEvent event) {
        refreshScope.refreshAll();
    }
}
```

## 常见问题排查

1. **Bugs 刷新后配置未生效**
   - 检查 `@RefreshScope` 是否标注在使用配置的 Bean 上（注意：@RefreshScope 只对加了该注解的 Bean 有效，其依赖的其他 Bean 不会重建）
   - `@ConfigurationProperties` 的类只需标注 `@RefreshScope` 即可刷新，但已经在使用的 Bean 不会自动重新注入——需要确保引用方也标注 @RefreshScope
   - 检查 Control Bus 端口（`spring.cloud.bus.id`）是否在多个服务间唯一

2. **事件未传播到其他服务**
   - 检查所有服务是否连接到同一个消息中间件实例（不同的 RabbitMQ vhost 或 Kafka topic 会导致隔离）
   - 检查 `spring.cloud.bus.destination` 配置是否一致（默认 `springCloudBus` 即可）

3. **Bus refresh 导致启动过慢**
   - Bus 客户端启动时会连接到消息中间件，如果中间件不可用，启动会卡住。可通过 `spring.cloud.bus.enabled=false` 禁用非必要实例的 Bus 功能
   - 建议配置 `spring.cloud.bus.env.enabled=true`（默认开启），允许 Bus 端点只在特定 profile 下启用

4. **RabbitMQ 配置正确但连接失败**
   - 检查防火墙和端口（5672 是 AMQP 端口，15672 是管理控制台——两者不要混淆）
   - 验证 RabbitMQ 的 vhost 和用户权限
