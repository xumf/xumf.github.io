---
title : "Spring 三级缓存"
date : "2025-01-30T22:39:17+08:00"
tags : ["Spring Refresh","Spring"]
---

## 三级缓存
在 Spring 框架中，**三级缓存**机制用于解决单例 Bean 的循环依赖问题。当两个或多个单例 Bean 相互依赖时，Spring 通过三级缓存来确保它们能够相互引用，避免了循环依赖倒置的错误。

### 三级缓存的组成：
> Spring的DefaultSingletonBeanRegistry类三级缓存对象

```java
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```
1. **一级缓存（****singletonObjects****）**：存放完全初始化的单例 Bean 实例。当请求一个 Bean 时，Spring 首先从一级缓存中查找，如果存在，则直接返回实例。
2. **二级缓存（****earlySingletonObjects****）**：存放尚未完全初始化的单例 Bean 实例，通常是指构造函数已执行完毕，但属性尚未填充的 Bean。当 Bean 的构造函数执行完毕后，但属性尚未填充的 Bean。当 Bean 的构造函数执行完毕后，Spring 会将其放入二级缓存中，以便在依赖注入过程中，其它 Bean 可以获取该 Bean 的早期引用，从而解决循环依赖。
3. **三级缓存（****singletonFactories****）**：存放用于创建单例 Bean 的工厂对象（ObjectFactory）。当 Bean 正在创建过程中，且需要提前暴露其引用时，Spring 会将用于创建该 Bean 的工厂对象放入三级缓存中。其它 Bean 在依赖注入时，可以通过三级缓存获取该工厂对象，从而获取正在创建中的 Bean 的引用。

### 三级缓存解决下循环依赖的流程
1. **实例化 Bean**：当 Spring 需要创建一个 Bean 时，首先会检查一级缓存中是否已有该 Bean 的实例。如果没有，则开始实例化该 Bean。
2. **放入三级缓存**：在实例化过程中，Spring 会将用于创建该 Bean 的工厂对象放入三级缓存中。这样，其它 Bean 在依赖注入时，可以通过三级缓存获取该工厂对象，从而获取到正在创建中的 Bean 的引用。
3. **放入二级缓存**：当 Bean 的构造函数执行完毕后，Spring 会将其放入二级缓存中。此时，其它 Bean 可以从二级缓存中获取到该 Bean 的早期引用。
4. **放入一级缓存**：当 Bean 的属性填充和初始化方法执行完毕后，Spring 会将其放入一级缓存，表示该 Bean 已完全初始化。此时，其它 Bean 可以从一级缓存中获取到该 Bean 的实例。

### 为什么需要三级缓存？
虽然二级缓存可以解决大部分循环依赖问题，但在涉及 AOP（面向切面编程）时，三级缓存显得尤为重要。在 AOP 代理的情况下，Bean 的创建过程需要通过代理工厂来生成代理对象。三级缓存缓存中的工厂对象正是用于创建这些代理对象的。如果没有三级缓存，Spring 将无法在 Bean 创建过程中提前暴露代理对象，从而导致循环依赖无法解决。

通过引用三级缓存，Spring 能够在 Bean 创建过程中提前暴露代理对象，确保循环依赖能够正确解决。这使得 Spring 在处理复杂的依赖关系时，能够保持高效和灵活。

总之，Spring 的三级缓存机制通过合理的缓存策略，确保了单例 Bean 在存在循环依赖和 AOP 代理的情况下，能够正确地相互引用，避免了循环依赖导致的错误。