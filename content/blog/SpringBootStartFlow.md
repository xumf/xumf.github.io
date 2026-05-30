---
title : "SpringBoot 启动流程"
date : "2023-05-09T23:36:22+08:00"
tags : [
  "Spring",
  "SpringBoot"
]

---

Spring Boot 的启动入口是 `SpringApplication.run()`，其核心流程可概括为三个阶段：**准备阶段 → 刷新上下文 → 启动完成后的回调**。

## 一、整体启动流程

```plain text
1. SpringApplication.run() 入口
2. 准备阶段：
   ├─ 推断应用类型（Reactive / Servlet）
   ├─ 加载所有 SpringApplicationRunListener
   ├─ 构建并准备 Environment（解析 application.yml/properties）
   └─ 打印 Banner
3. 创建并准备 ApplicationContext：
   ├─ 创建上下文实例（AnnotationConfigServletWebServerApplicationContext）
   ├─ 执行 ApplicationContextInitializer
   ├─ 注册启动参数中的主配置类（@SpringBootApplication 标注的类）
   └─ 调用 refresh() 方法 → 进入 Spring 容器核心流程
4. 刷新后回调：
   ├─ 执行 CommandLineRunner / ApplicationRunner
   └─ 启动内嵌 Web 容器（Tomcat / Jetty / Undertow）
```

`refresh()` 方法是 Spring 容器初始化的核心，由 `AbstractApplicationContext.refresh()` 定义，其中 13 个步骤依次执行。

## 二、refresh() 核心步骤

`refresh()` 中的关键步骤（按顺序）：

1. **prepareRefresh()**：设置容器状态，初始化属性源和早期事件发布器
2. **obtainFreshBeanFactory()**：刷新 BeanFactory，解析 Bean 定义（从配置类、XML、@ComponentScan 扫描结果）
3. **prepareBeanFactory()**：配置 BeanFactory 的类加载器、SpEL 解析器、属性编辑器，注册 Aware 接口回调需要的 Bean
4. **postProcessBeanFactory()**：子类扩展点（Web 容器在此注册 ServletContextAware 处理器）
5. **invokeBeanFactoryPostProcessors()**：调用所有 BeanFactoryPostProcessor。最关键的是 **ConfigurationClassPostProcessor**，它解析 @Configuration、@ComponentScan、@Import、@Bean 并注册所有 Bean 定义
6. **registerBeanPostProcessors()**：注册 BeanPostProcessor。此时只注册不执行——真正执行时机在 bean 实例化阶段触发
7. **initMessageSource()**：初始化国际化组件
8. **initApplicationEventMulticaster()**：初始化事件广播器
9. **onRefresh()**：创建并启动内嵌 Web 容器（Tomcat 在此启动）
10. **registerListeners()**：注册所有的 ApplicationListener
11. **finishBeanFactoryInitialization()**：实例化所有非懒加载的单例 Bean——这是**启动最耗时**的阶段
12. **finishRefresh()**：发布 ContextRefreshedEvent 事件

## 三、核心 BeanPostProcessor 详解

BeanPostProcessor 在 bean 实例化过程的两个时机触发：**初始化前（BeforeInitialization）** 和 **初始化后（AfterInitialization）**。下面列出 Spring Boot 自动装配中最常用的几个：

### 依赖注入相关
1. **AutowiredAnnotationBeanPostProcessor**
   * 处理 @Autowired、@Value、@Inject
   * postProcessBeforeInitialization：解析注入点并注入依赖
   * 源码：`org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor`

2. **CommonAnnotationBeanPostProcessor**
   * 处理 JSR-250 注解：@Resource、@PostConstruct、@PreDestroy
   * @Resource 按名称优先注入，与 @Autowired 的 Type 优先策略不同——混用时注意
   * 源码：`org.springframework.context.annotation.CommonAnnotationBeanPostProcessor`

### 生命周期回调相关
3. **InitDestroyAnnotationBeanPostProcessor**
   * 处理 @PostConstruct 和 @PreDestroy
   * 与 CommonAnnotationBeanPostProcessor 的职责有重叠——InitDestroyAnnotationBeanPostProcessor 更底层，CommonAnnotationBeanPostProcessor 扩展了对 @Resource 的支持

### Aware 接口注入
4. **ApplicationContextAwareProcessor**
   * 处理 ApplicationContextAware、EnvironmentAware、ResourceLoaderAware 等
   * postProcessBeforeInitialization 阶段注入，早于 @Autowired

### 占位符解析
5. **EmbeddedValueResolverAwareBeanPostProcessor**
   * 处理 EmbeddedValueResolverAware 接口，注入 StringValueResolver
   * 用于解析 `${property}` 占位符和 `#{spEL}` 表达式

### AOP 代理
6. **InfrastructureAdvisorAutoProxyCreator**
   * 为符合条件的 Bean 生成 AOP 代理
   * postProcessAfterInitialization 阶段触发：先有 Bean 实例，再通过代理增加切面逻辑
   * 源码：`org.springframework.aop.framework.autoproxy. InfrastructureAdvisorAutoProxyCreator`

### 异常转换
7. **PersistenceExceptionTranslationPostProcessor**
   * 将持久化层异常（JPA、Hibernate）转换为 Spring DataAccessException
   * 通常提前注册 BeanPostProcessor 以处理所有持久化 Bean

### 事件与注解驱动
8. **EventListenerMethodProcessor**
   * 处理 @EventListener，注册事件监听方法
   * 源码：`org.springframework.context.event.EventListenerMethodProcessor`

9. **ScheduledAnnotationBeanPostProcessor**
   * 处理 @Scheduled，注册定时任务
   * 与 @Async 由不同处理器管理：@Async 由 AsyncAnnotationBeanPostProcessor 处理

### Import 扩展
10. **ImportAwareBeanPostProcessor**
    * 处理 @Import 导入的类中实现 ImportAware 接口的 Bean
    * postProcessBeforeInitialization 阶段注入导入元数据

## 四、BeanFactoryPostProcessor

1. **ConfigurationClassPostProcessor**
   * **最重要的 BeanFactoryPostProcessor**，负责解析 @Configuration 类
   * 处理 @ComponentScan（包扫描）、@Import（导入配置类）、@Bean（方法级别 Bean 定义）
   * 在 invokeBeanFactoryPostProcessors() 阶段执行——比任何 BeanPostProcessor 都早
   * 没有它，所有注解配置的 Bean 都不会被注册

## 五、常见问题排查

1. **启动慢怎么办？**
   * 排查点：finishBeanFactoryInitialization 阶段耗时最长，检查是否有大量懒加载泄漏、@ComponentScan 覆盖过多包路径
   * 启用 lazy-initialization（`spring.main.lazy-initialization=true`）可加速启动，但注意首次请求变慢
2. **Bean 初始化异常如何排查？**
   * 开启 `debug=true` 查看 ConditionEvaluationReport（自动配置条件评估报告）
   * 关键日志关键词：`UnsatisfiedDependencyException`、`BeanCreationException`、`NoSuchBeanDefinitionException`
3. **AOP 代理不生效？**
   * 检查类是否被 final 修饰（CGLIB 无法代理 final 类）
   * 检查同一个类内部方法调用 AOP 代理问题（自调用不走代理）
4. **@PostConstruct 与 afterPropertiesSet 的执行顺序？**
   * `InitializingBean.afterPropertiesSet()` → `@PostConstruct` → 自定义 `init-method`
   * 顺序：InitializingBean → BeanPostProcessor.postProcessBeforeInitialization → @PostConstruct → afterPropertiesSet → init-method → BeanPostProcessor.postProcessAfterInitialization
5. **ConfigurationClassPostProcessor 的常见错误**
   * @ComponentScan 未生效：检查启动类的位置是否在根包
   * @Bean 方法被 final 修饰：CGLIB 无法重写 final 方法
   * @Import 导入的配置类重复：Spring 5.2+ 支持 @Import 去重，但注意 @Import 和 @ComponentScan 同时扫描到同一个类会导致重复注册
