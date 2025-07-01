---
title: "Spring Cloud Bus"
date: 2024-01-05T13:42:22+08:00
tags: ["Spring Cloud Bus", "Spring Cloud"]
---

## Spring Cloud Bus 的核心概念
### （1）什么是 Spring Cloud Bus？
* Spring Cloud Bus 是一个分布式事件总线，用于在微服务之间传播状态变化（如配置更新）
* 它基于消息中间件（如 RabbitMQ、Kafka）实现，支持全局配置刷新、服务状态同步等功能

### （2）Spring Cloud Bus 的组成
* 消息中间件：RabbitMQ 或 Kafka，负责传递事件
* 事件生产者：Config Server 或其他微服务，发送事件
* 事件消费者：Config Client 或其他微服务，接收事件并执行相应操作

### （3）Spring Cloud Bus 的特点
* 全局配置刷新：通过 **/actuator/bus-refresh **端点触发全局配置刷新
* 服务状态同步：通过事件传播实现服务状态同步
* 多消息中间件支持：支持 RabbitMQ 和 Kafka

## Spring Cloud Bus 的工作原理
### （1）事件传播流程
1. 事件生产者（如 Config Server）发送事件到消息中间件
2. 消息中间件将事件广播给所有订阅者
3. 事件消费者（如 Config Client）接收到事件并执行相应操作

### （2）全局配置刷新流程
1. Config Server 接收到 **/actuator/bus-refresh** 请求
2. Config Server 通过消息中间件广播一个刷新事件
3. 所有 Config Client 接收到刷新事件
4. Config Client 调用 RefreshScope.refreshAll()，刷新标记为 **@RefreshScope** 的 Bean

## Spring Cloud Bus 配置
### （1）Kafka 配置
* 如果你使用 Kafka 作为消息中间件，可以这样配置：

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: my-group
```
### （2）启用 Spring Cloud Bus
* 在 pom.xml 中添加依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId> <!-- RabbitMQ -->
    <!-- 或者 -->
    <artifactId>spring-cloud-starter-bus-kafka</artifactId> <!-- Kafka -->
</dependency>
```
### （3）暴露 Actuator 端点
* 在 **application.yml** 中暴露 **/actuator/bus-refresh** 端点：

```xml
management:
  endpoints:
    web:
      exposure:
        include: bus-refresh
```
## Spring Cloud Bus 的使用场景
### （1）全局配置刷新
* 通过 **/actuator/bus-refresh** 端点触发全局配置刷新

* 示例：

```bash
curl -X POST http://localhost:8888/actuator/bus-refresh
```
### （2）服务状态同步
* 通过事件传播实现服务状态的同步
* 示例：当某个服务的状态发生变化时，通过 Spring Cloud Bus 通知其他服务

### （3）自定义事件
* 通过 Spring Cloud Bus 发送和接收自定义事件
* 示例：

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
## Spring Cloud Bus 的源码解析
### （1）BusRefreshEndpoint 类
* BusRefreshEndpoint 是 Spring Cloud Bus 提供的端点，用于触发全局刷新
* 源码：

```java
@Endpoint(id = "bus-refresh")
public class BusRefreshEndpoint {

    private final ApplicationEventPublisher publisher;

    public BusRefreshEndpoint(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    @WriteOperation
    public void refresh() {
        // 发布刷新事件
        publisher.publishEvent(new RefreshRemoteApplicationEvent(this, null, null));
    }
}
```
### （2）RefreshRemoteApplicationEvent 类
* RefreshRemoteApplicationEvent 是 Spring Cloud Bus 的事件类，用于表示刷新事件
* 源码：

```java
public class RefreshRemoteApplicationEvent extends RemoteApplicationEvent {
    public RefreshRemoteApplicationEvent(Object source, String originService, String destinationService) {
        super(source, originService, destinationService);
    }
}
```
### （3）RefreshListener 类
* RefreshListener 是 Spring Cloud Bus 的监听器，负责处理刷新事件
* 源码：

```java
public class RefreshListener implements ApplicationListener<RefreshRemoteApplicationEvent> {

    private final RefreshScope refreshScope;

    public RefreshListener(RefreshScope refreshScope) {
        this.refreshScope = refreshScope;
    }

    @Override
    public void onApplicationEvent(RefreshRemoteApplicationEvent event) {
        // 触发刷新
        refreshScope.refreshAll();
    }
}
```
