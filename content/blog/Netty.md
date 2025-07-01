---
title: "Netty知识"
date: 2022-06-22T13:16:53+08:00
tags: ["Netty", "Server"]
---

## 一、Netty 线程模型与 Reactor 模式
1. **主从 Reactor 多线程模型**

Netty 采用主从 Reactor 模型，核心组件为`EventLoopGroup` ：

   * **BossGroup(主 Reactor)**：负责监听客户端连接，通过`Selector` 将新连接注册到 WorkerGroup 中的某个`EventLoop` 。
   * **WorkerGroup(从 Reactor)**：处理已建立连接的 I/O 事件（入读写）及业务逻辑，每个`EventLoop` 绑定一个线程，通过轮询`Selector` 监听多个`Channel` 事件，避免线程切换开销。

主从 Reactor 配置示例

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1); // 主Reactor，一般只需1个线程处理连接
EventLoopGroup workerGroup = new NioEventLoopGroup(); // 从Reactor，默认线程数=CPU核心*2

ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
 .channel(NioServerSocketChannel.class)
 .childHandler(new ChannelInitializer<SocketChannel>() {
     @Override
     protected void initChannel(SocketChannel ch) {
         ch.pipeline().addLast(new MyBusinessHandler());
     }
 });
```
调优点：
      * `bossGroup` 线程数通常为1（除非需要多端口监听）
      * `workerGroup` 线程数根据业务类型调整：
         * CPU 密集型：线程数 ≈ CPU 核心数
         * I/O 密集型：线程数 ≈ CPU 核心数 \* 2
      * 使用 Epoll 模式（Linux 环境）：

```java
EventLoopGroup group = new EpollEventLoopGroup();
```
2. **无锁化串行设计**

Netty 通过 `ChannelPipeline` 将事件处理任务分配给固定的`EventLoop` 线程执行，保证同一连接的所有操作在同一个线程内完成，避免多线程竞争，减少锁的使用。

---
## 二、零拷贝(Zero-Copy)的深度实现
1. **操作系统级零拷贝**

使用`FileRegion` 和`transferTo()` 方法，文件数据直接从内核缓冲区通过 DMA 传输到 Socket 缓冲区，无需用户态与内核态的数据拷贝。

2. **Netty 层的优化**
   * **CompositeByteBuf**：合并多个`ByteBuf` 为逻辑视图，避免物理内存拷贝。
   * **直接内存(Direct Buffer)**：分配堆外内存，避免 JVM 堆与 Socket 缓冲区间的拷贝，但需要手动管理内存释放。
3. **示例：**
   1. **FileRegion 实现文件传输(零拷贝)**

```java
File file = new File("largefile.zip");
FileInputStream fis = new FileInputStream(file);
FileRegion region = new DefaultFileRegion(fis.getChannel(), 0, file.length());

ctx.channel().writeAndFlush(region).addListener(future -> {
    fis.close();
    if (!future.isSuccess()) {
        future.cause().printStackTrace();
    }
});
```
   2. **CompositeByteBuf 合并缓冲区**

```java
ByteBuf header = Unpooled.copiedBuffer("Header", CharsetUtil.UTF_8);
ByteBuf body = Unpooled.copiedBuffer("Body", CharsetUtil.UTF_8);

CompositeByteBuf compositeBuf = Unpooled.compositeBuffer();
compositeBuf.addComponents(true, header, body); // 逻辑合并，无内存拷贝
```
**调优点**：

   * 堆外内存（Direct Buffer）分配：

```java
ByteBuf directBuf = ByteBufAllocator.DEFAULT.directBuffer(1024);
```
   * 监控 Direct Buffer 使用量（避免 OOM）：

```bash
-Dio.netty.maxDirectMemory=256M  # JVM参数限制最大堆外内存
```
---
## 三、内存管理与性能优化
1. **内存池(PooledByteBufAllocator)**

Netty 通过对象池复用`ByteBuf` ，减少频繁的内存分配与 GC 压力。直接内存池采用 Buddy 算法管理内存块，提升分配效率。

2. **内存泄漏检测机制**

通过`ReferenceCounted` 接口实现引用计数，结合`ResourceLeakDetector` 监控未释放的`ByteBuf` ，防止内存泄漏。

   * **内存泄漏检测配置**

```java
// 设置泄漏检测级别（生产环境建议SIMPLE）
ResourceLeakDetector.setLevel(ResourceLeakDetector.Level.PARANOID);

// 引用计数示例
ByteBuf buf = ByteBufAllocator.DEFAULT.buffer();
buf.retain(); // 增加引用计数
buf.release(); // 减少引用计数，当计数=0时释放内存
```
   * **内存池配置**

```java
// 服务端启用内存池
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
```
   * **调优参数**
      * `-Dio.netty.allocator.type=pooled` 强制启用内存池
      * `-Dio.netty.allocator.maxOrder=5` 调整内存块大小（默认11，对应8MB块）
      * `-Dio.netty.threadLocalDirectBufferSize=0` 禁用 ThreadLoad 缓存（大内存场景）

---
## 四、TCP 粘包/拆包解决方案
1. **协议设计**
   * **定长协议**：`FixedLengthFrameDecoder` 处理固定长度消息。
   * **分隔符协议**：`DelimiterBasedFrameDecoder` 按自定义分隔符（如`\n` ）拆分数据。
   * **头部长度字段协议**：`LengthFieldBasedFrameDecoder` 解析消息头中的长度字段，动态拆分数据。
2. **自定义编解码器**

结合`MessageToByteEncoder` 和`ByteToMessageDecoder` 实现私有协议，例如 Dubbo 的协议头设计。

```java
// 协议格式：4字节长度 + 数据
public class CustomDecoder extends LengthFieldBasedFrameDecoder {
    public CustomDecoder() {
        super(1024 * 1024, 0, 4, 0, 4); // 最大长度1MB，长度字段偏移0
    }

    @Override
    protected Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
        ByteBuf frame = (ByteBuf) super.decode(ctx, in);
        if (frame == null) return null;
        int dataLength = frame.readInt();
        byte[] data = new byte[dataLength];
        frame.readBytes(data);
        return new String(data, StandardCharsets.UTF_8);
    }
}

// 添加解码器到Pipeline
pipeline.addLast(new CustomDecoder());
pipeline.addLast(new MyBusinessHandler());
```
---
## 五、心跳机制与长连接管理
1. **IdleStateHandler**

监控读写空闲时间，触发`IdleStateEvent` 事件，结合业务实现心跳包发送或断连处理。例如：

```java
pipeline.addLast(new IdleStateHandler(30, 0, 0, TimeUnit.SECONDS));
pipeline.addLast(new HeartbeatHandler()); // 自定义处理空闲事件
```
2. **TCP Keep-Alive 与应用层心跳**

TCP 层 Keep-Alive 间隔较长（默认2小时），应用层心跳（如每30秒一次）更灵活，可快速检测网络异常。

---
## 六、高性能背后的设计哲学
1. **异步事件驱动模型**

基于NIO的非阻塞 I/O，通过`ChannelFuture` 实现异步回调，避免线程阻塞。

2. **责任链模式(ChannelPipeline)**

将编解码、业务逻辑等`ChannelHanlder` 串联成链，事件按顺序传播，支持动态扩展。

3. **线程局部变量(FastThreadLocal)**

Netty 优化了 JDK 的`ThreadLocal`，减少哈希冲突，提升访问速度。

---
## 七、高性能调优参数
1. **Linux 系统参数调优**

```bash
# 最大文件描述符数
ulimit -n 1000000

# TCP缓冲区设置
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"
sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"
```
2. **Netty 关键参数**

```java
// 服务端配置
bootstrap.option(ChannelOption.SO_BACKLOG, 1024) // 等待连接队列大小
         .option(ChannelOption.SO_REUSEADDR, true) // 端口复用
         .childOption(ChannelOption.TCP_NODELAY, true) // 禁用Nagle算法
         .childOption(ChannelOption.SO_KEEPALIVE, true); // 启用TCP Keep-Alive
```
3. **JVM 参数调优**

```bash
# 堆外内存限制
-XX:MaxDirectMemorySize=512m

# GC优化（G1为例）
-XX:+UseG1GC 
-XX:MaxGCPauseMillis=200 
-XX:InitiatingHeapOccupancyPercent=30
```
---
## 八、高并发场景实战
1. **百万连接优化**：
   * **调整 Epoll 事件触发模式**

```java
EpollEventLoopGroup group = new EpollEventLoopGroup();
bootstrap.channel(EpollServerSocketChannel.class); // 使用Epoll边缘触发
```
   * **减少上下文切换**：

```java
workerGroup.setIoRatio(70); // I/O任务占比70%，业务任务30%
```
   * **连接数监控**

```java
ChannelGroup allChannels = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);
allChannels.add(channel); // 维护所有活跃Channel
```
2. **异步处理耗时操作**

```java
// 业务Handler中提交异步任务
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    executorService.execute(() -> {
        Object result = heavyCompute(msg); // 耗时操作
        ctx.writeAndFlush(result);
    });
}
```
## 九、重要知识点
1. **如何避免 Epoll 空轮询 Bug？**

Netty 通过重建 Selector 解决 JDK NIO 的空轮询问题：当 Selector 空轮询次数超过阈值，重新注册 Channel 到新 Selector。

2. **Channel.write() 与 ChannelHandlerContext.write() 的却别？**

`Channel.write()` 从 Pipeline 尾部开始传播，`ChannelHandlerContext.write()` 从当前 Handler 的下一个节点开始，影响事件处理路径。

3. **如何实现百万级长连接？**
   * 调整系统参数：最大文件描述符数、TCP 缓冲区大小。
   * 是用 Epoll 边缘触发模式（`EpollEventLoopGroup` ）
   * 优化内存管理，避免频繁 GC。
4. **内存泄漏排查**
   * 使用`-Dio.netty.leakDetection.level=paranoid` 开启详细日志
   * 通过`jmap -histo:live <pid>` 查看 Direct Buffer 数量
5. **CPU 100%排查**
   * `top -Hp <pid>` 找到高 CPU 线程
   * `jstack <pid>` 查看线程堆栈，定位 EventLoop 阻塞点
6. **网络拥塞分析**
   * 使用`netstat -ant|awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}` 监控 TCP 状态
   * 通过 Wireshark 抓包分析粘包/拆包