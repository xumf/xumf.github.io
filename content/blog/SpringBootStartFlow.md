---
title : "SpringBoot 启动流程"
date : "2023-05-09T23:36:22+08:00"
tags : [
  "Spring",
  "SpringBoot"
]

---

### 核心的 BeanPostProcessor
1. `AutowiredAnnotationBeanPostProcessor`
   * **作用**：处理`@Autowired` 、`@Value` 、`@Inject` 等注解，实现依赖注入。
   * **触发时机**：
      * **postProcessBeforeInitialization**：在 Bean 初始化之前，解析并注入依赖
      * **postProcessAfterInitialization**：在 Bean 初始化之后，通常不执行额外逻辑
   * **源码类**：`org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor`
2. `CommonAnnotationBeanPostProcessor` 
   * **作用**：处理 JSR-250 标准注解，如 `@Resource` 、`@PostConstruct` 、`@PreDestory` 
      * `@Resource` ：按名称或类型注入 Bean
      * `@PostConstruct` ：在 Bean 初始化后执行
      * `@PreDestory` ：在 Bean 销毁前执行
   * **触发时机**：
      * **postProcessBeforeInitialization**：在 Bean 初始化之前，处理 `@PostConstruct` 注解的方法
      * **postProcessAfterInitialization**：在 Bean 初始化之后，处理 `@PreDestroy` 注解的方法
   * **源码类**：`org.springframework.context.annotation.CommonAnnotationBeanPostProcessor` 
3. `RequiredAnnotationBeanPostProcessor`
   * **作用**：检查 `@Required`  注解标注的属性是否已注入，未注入则抛出异常
   * **触发时机**：
      * **postProcessoBeforeInitialization**：在 Bean 初始化前，检查 `@Required` 属性是否已注入
   * **源码类**：`org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor`
4. `ApplicationContextAwareProcessor`
   * **作用**：处理 `ApplicationContextAware` 接口，向 Bean 注入 `ApplicationContext` 
   * **触发时机**：
      * **postProcessorBeforeInitialization**：在 Bean 初始化之前，调用 `setApplicationContext()` 方法
   * **源码类**：`org.springframework.context.support.ApplicationContextAwareProcessor`
5. `InitDestroyAnnotationBeanPostProcessor` 
   * **作用**：处理 `@PostConstruct` 和 `@PreDestory` 注解，分别在 Bean 初始化和销毁时执行
   * **触发时机**：
      * **postProcessBeforeInitialization** ：在 Bean 初始化之前，执行 `@PostConstruct`  注解的方法
      * **postProcessAfterInitialization**：在 Bean 初始化之后，注册 `@PreDestory` 注解的方法，以便在 Bean 销毁时执行
6. `InfrastructureAdvisorAutoProxyCreator`
   * **作用**：为 Bean 创建代理，支持 AOP （面向切面编程）
   * **触发时机**：
      * **postProcessAfterInitialization**：在 Bean 初始化之后，根据切面规则生成代理对象
   * **源码类**：`org.springframework.aop.framework.autoproxy.InfrastructureAdvisorAutoProxyCreator`
7. `PersistenceExceptionTranslationPostProcessor` 
   * **作用**：将持久化层异常（如 JPA、Hibernate 异常）转换为 Spring 的统一异常
   * **触发时机**：
      * **postProcessorBeforeInitialization**：在 Bean 初始化之前，注册异常翻译器
   * **源码类**：`org.springframework.dao.annotation.PersistenceExceptionTranslationPostProcessor`
8. `EventListenerMethodProcessor`
   * **作用**：处理 `@EventListener` 注解，将标注的方法注册为事件监听器
   * **触发时机**：
      * **postProcessAfterInitialization**：在 Bean 初始化之后，扫描并注册 `@EventListener` 方法
   * **源码类**：`org.springframework.context.event.EventListenerMethodProcessor`
9. `ScheduleAnnotationBeanPostProcessor`
   * **作用**：处理 `@Schedule` 注解，将标注的方法注册为定时任务
   * **触发时机**：
      * **postProcessAfterInitialization**：在 Bean 初始化之后，解析 `@Scheduled` 注解并注册定时任务
      * **源码类**：`org.springframework.scheduling.annotation.ScheduledAnnotationBeanPostProcessor`
10.  `ImportAwareBeanPostProcessor`
   * **作用**：处理 `@Import` 注解导入的类，支持 `ImportAware` 接口
   * **触发时机**：
      * **postProcesssBeforeInitializtion**：在 Bean 初始化之前，调用 `setImportMetadata()` 方法
      * **源码类**：`org.springframework.context.annotation.ImportAwareBeanPostProcessor`
11.  `EmbeddedValueResolverAwareBeanPostProcessor`
   * **作用**：
      * 处理 `EmbeddedValueResolverAware` 接口，向 Bean 注入 `StringValueResolver`。
      * `StringValueResolver` 用于解析占位符（如 `${property}`）和 SpEL 表达式（如 `#{expression}`）
   * **触发时机**：
      * **postProcessBeforeInitialization**：在 Bean 初始化之前，调用 `setEmbeddedValueResolver()` 方法
   * **源码类**：`org.springframework.context.support.EmbeddedValueResolverAwareBeanPostProcessor`



### 核心的BeanFactoryPostProcessor
1. `ConfigurationClassPostProcessor` 
   * **作用**：处理 `@Configuration` ，解析 `@ComponentScan` 、`@Import` 、`@Bean` 等注解
   * **触发时机**：
      * 在 `BeanFactoryPostProcessor` 阶段执行，负责加载配置类并注册 Bean 定义
      * 不属于 `BeanPostProcessor` 的执行时机，而是在容器刷新时提前执行
   * **源码类**：`org.springframework.context.annotation.ConfigurationClassPostProcessor`


