---
title: "§0 Netty 核心概念 & vs Tomcat 优势 — mu-server 背景补充"
category: synthesis
tags: [java, netty, tomcat, framework, background, reactor, event-loop, 2.4.2, sub-page]
sources:
  - "Netty 官方文档 (https://netty.io/)"
  - "Netty in Action — Norman Maurer (Manning, 2015)"
  - "Tomcat 官方文档 (https://tomcat.apache.org/)"
  - "Webflux vs Tomcat vs Netty benchmark (ruslanys/sample-spring-boot-netty, 2024)"
summary: "mu-server 的 Netty 底层背景：Netty 4.x 核心概念（EventLoop / ChannelPipeline / ByteBuf / Future）+ Tomcat 线程模型对比 + 为什么 mu-server 选 Netty。作为 mu-server 分析的背景补充知识。"
provenance:
  extracted: 0.92
  inferred: 0.05
  ambiguous: 0.03
  base_confidence: 0.90
lifecycle: reviewed
lifecycle_changed: 2026-09-12
created: 2026-09-12
updated: 2026-09-12
---

# §0 Netty 核心概念 & vs Tomcat 优势 — mu-server 背景补充

> **为什么读这章**: mu-server 是基于 Netty 的 Web 服务器。要理解 §2 协议层 / §3 抽象层 / §4 分发层 / §8 线程模型的设计动机，先要懂 Netty 4.x 的核心概念和线程模型。本章对比 Netty 和 Tomcat（Java 生态最主流的 Web 容器），说明 mu-server 为何选 Netty。

---

## 0.1 为什么需要 Netty（Java NIO 的痛点）

直接用 JDK NIO 写网络应用很难。Netty 解决的几个核心问题：

| 痛点 | 说明 |
|---|---|
| **Selector 状态管理复杂** | Selector 的 select() / selectedKeys() / iterator() 状态机繁琐 |
| **ByteBuffer 容量固定** | 一旦分配不能扩容；flip() / rewind() / clear() 容易出错 |
| **JDK epoll 空转 bug** | Linux epoll 在 select() 返回 0 后可能假唤醒，CPU 100% |
| **TCP 拆包/粘包** | TCP 是字节流协议，需要 LengthFieldBasedFrameDecoder 等处理 |
| **没有 Codec 框架** | HTTP/HTTPS/WebSocket/Protobuf 要自己实现编解码 |
| **多线程同步陷阱** | Selector + 多线程 worker 容易死锁 |

Netty 把这些都封装好了：基于 Multi-threaded Reactor Pattern + 内置 codec 框架 + 池化 ByteBuf + 修复 epoll bug。

---

## 0.2 Netty 4.x 核心概念

### 0.2.1 Channel & ChannelPipeline

- **Channel**: 一个网络连接的抽象（TCP socket / UDP / etc.）
- **ChannelPipeline**: 每个 Channel 一个责任链（filter chain）
- **ChannelHandler**: 业务处理单元（用户编写的）

事件流向：
- **入站**（Inbound）：Head → Tail（网络 → 业务）
- **出站**（Outbound）：Tail → Head（业务 → 网络）

```java
pipeline.addLast(new SslHandler(sslCtx));      // 1. TLS 解密
pipeline.addLast(new HttpServerCodec());       // 2. HTTP 编解码
pipeline.addLast(new MyBusinessHandler());    // 3. 业务逻辑
```

**关键设计**：handler 可以在运行时动态增删（on-the-fly composition）。

### 0.2.2 EventLoop（Netty 最核心的概念）

- **EventLoop = 1 个线程 + 1 个 Selector + 任务队列**
- 1 个 Channel 绑定 1 个 EventLoop（Round-Robin 分配，channel 生命周期不变）
- EventLoop **单线程**处理该 channel 的所有事件 → **无锁并发**！
- **关键约束**：handler 不能阻塞（阻塞 = 阻塞该 channel 所有 I/O）
- 慢操作必须 offload 到独立 `EventExecutorGroup`

Netty 4 重要改动（vs Netty 3）：**入站 + 出站事件都在同一 IO thread** 处理（Netty 3 出站在调用线程，导致并发模型复杂）。这是 4.x 简化的关键。

### 0.2.3 ByteBuf（替代 JDK ByteBuffer）

| 特性 | JDK ByteBuffer | Netty ByteBuf |
|---|---|---|
| 容量 | 固定（allocate 后不能改） | **动态扩容** |
| 指针 | 单 position（需手动 flip） | **双指针**（readerIndex + writerIndex） |
| 内存管理 | 无 | **池化** + **引用计数** |
| 0 拷贝 | 无 | CompositeByteBuf 聚合多个 buffer 不复制 |
| 释放 | 无 | `release()` 自动归还到池 |

引用计数关键：`SimpleChannelInboundHandler` 自动 release 入站 ByteBuf，避免最常见的 Netty 内存泄漏。

### 0.2.4 Future / Promise（异步编程）

```java
ChannelFuture future = channel.writeAndFlush(response);
future.addListener(f -> {
    if (f.isSuccess()) System.out.println("Send OK");
    else f.cause().printStackTrace();
});
```

- `ChannelFuture` 替代 JDK `Future`（支持 listener）
- **优先用 `addListener()`**（不阻塞）
- `sync()` 同步等待（仅在确认线程安全时用）
- `Promise` 是可写的 Future（用于跨线程传值）

### 0.2.5 编解码器（CODEC）

| 类型 | 类 | 用途 |
|---|---|---|
| 字节 → POJO | `ByteToMessageDecoder` | 拆包 + 解码 |
| POJO → 字节 | `MessageToByteEncoder` | 编码 + 序列化 |
| 协议 codec | `HttpServerCodec` / `Http2FrameCodec` / `WebSocket*` | 内置协议 |

mu-server 自己实现 HTTP/1 (`Http1Connection`) + HTTP/2 (`Http2Connection`) 编解码而非用 Netty 内置，是为了更细粒度的背压控制（见 §2.1 / §2.2）。

---

## 0.3 Netty 线程模型

### 0.3.1 bossGroup + workerGroup

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);     // 接收连接
EventLoopGroup workerGroup = new NioEventLoopGroup(N);   // 处理 I/O

ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
 .channel(NioServerSocketChannel.class)
 .childHandler(new ChannelInitializer<SocketChannel>() { ... });
```

| 角色 | 线程数 | 职责 |
|---|---|---|
| **bossGroup** | 1-2 | accept() 新连接，注册到 workerGroup |
| **workerGroup** | 默认 `min(16, CPU*2)` | 处理所有 I/O 事件（read/write）+ 用户 handler |

mu-server 的 `MuServerBuilder` 也用这套（详见 §4.2）。

### 0.3.2 关键设计决策（Netty 4 vs Netty 3）

| 决策 | Netty 3 | Netty 4 |
|---|---|---|
| 出站事件线程 | 调用线程（业务线程可写 socket） | **IO thread**（同入站） |
| 事件传递 | ChannelEvent 对象 | **直接方法调用**（减少 GC） |
| 线程模型 | Half-async | **Multi-threaded Reactor** |
| Future | ChannelFuture | ChannelFuture + addListener |

---

## 0.4 Tomcat 线程模型对比

### 0.4.1 传统 BIO（Tomcat 7-，现已 deprecated）

- Connector 默认 BIO（`protocol="HTTP/1.1"`）
- **一个连接一个线程**：acceptor 阻塞 accept → 分给 worker thread
- `ThreadPoolExecutor`（默认 `maxThreads=200`）
- 阻塞 I/O + 同步 Servlet API
- 简单但并发上限 ~200（线程池耗尽即拒绝）

### 0.4.2 NIO Connector（Tomcat 8+ 默认）

- `protocol="org.apache.coyote.http11.Http11NioProtocol"`
- **三层模型**：
  - **Acceptor**：1-N 线程，accept() 新连接（仍阻塞）
  - **Poller**：1-N 线程，Selector 多路复用读 socket
  - **Worker Pool**：N 线程，跑 Servlet.service()（同步阻塞）
- 并发能力提升（Poller 一个线程处理多个连接）
- **但 Servlet API 仍同步阻塞** → worker 线程仍按请求分配

### 0.4.3 Servlet 编程模型

- `HttpServletRequest` / `HttpServletResponse` 同步阻塞 API
- 业务线程 = Servlet.service() 线程
- Servlet 3.0+ 加了 `AsyncContext`（支持异步 Servlet）
- Servlet 5.0+ 支持 Jakarta EE 9 / 10

---

## 0.5 Netty vs Tomcat 对比

| 维度 | Tomcat | Netty |
|---|---|---|
| **I/O 模型** | NIO Connector（Poller 多路复用）| **NIO Reactor（全异步）** |
| **线程模型** | 一个请求一个 worker 线程 | **一个 EventLoop 处理多 channel** |
| **最大并发连接** | ~数千（受线程池限制）| **~数万-百万**（受 EventLoop 数限制） |
| **吞吐（实测）** | 8,822 req/s（wrk 12线程 400 连接）| **28,151 req/s** — **3.18x 更快** |
| **P50 延迟** | 42.26 ms | **12.09 ms**（3.5x 低）|
| **P99 延迟** | 275.51 ms | 215.51 ms |
| **编程模型** | **同步阻塞** Servlet | **异步事件驱动** ChannelHandler |
| **协议支持** | HTTP/HTTPS/HTTP2/WebSocket | **任意 TCP/UDP/自定义二进制协议** |
| **生态** | Spring 全家桶 / Jakarta EE | Dubbo / RocketMQ / Elasticsearch |
| **学习曲线** | **低**（标准化）| 高（事件驱动思维） |
| **典型场景** | 企业 Web / JSP / Spring MVC | 微服务 / RPC / 长连接 / 网关 / 嵌入式 |

**实测数据来源**：ruslanys/spring-boot-netty benchmark（wrk 12 线程 400 连接 30s，2 核 2GB KVM，JVM `-Xmx1024M -server`，endpoint `/api/user/current`）：

```
Tomcat          8,822 req/s   (baseline)
Undertow       10,090 req/s   (1.14x)
Spring Webflux 10,193 req/s   (1.16x)
Netty-based    28,151 req/s   (3.18x) ← Netty
```

**为什么 Netty 更快**：
1. 异步非阻塞无锁（EventLoop 单线程）
2. 池化 ByteBuf（少 GC）
3. 0 拷贝（CompositeByteBuf + FileRegion）
4. 减少上下文切换（线程数远低于 Tomcat worker pool）
5. CPU 利用率更高（无 idle 线程等 I/O）

---

## 0.6 为什么 mu-server 选 Netty

1. **嵌入式友好**
   - 直接 `new MuServerBuilder().start()`，不需 war 部署
   - 不需 Servlet 容器（Tomcat/Jetty/Undertow）
   - 启动 < 1 秒，适合 CLI 工具 / 桌面应用 / 微服务

2. **HTTP/1 + HTTP/2 + 自实现流控**
   - 避免 Netty 默认 HTTP/2 流控在大 body 场景的 deadlock
   - `Http2ConnectionFlowControl`（在 `Http2Connection` 内部）精细控制

3. **轻量**
   - 核心 248 文件 / 31,840 行（vs Tomcat 数千文件）
   - 一个 builder 启动 server，学习曲线低
   - 团队规模小可维护

4. **零 Servlet 依赖**
   - 现代 Jakarta REST 3.0+ (`@Path`/`@GET`/`@POST`) 替代 servlet
   - 干净的 `MuRequest` / `MuResponse` API（无 `HttpServletRequest` 历史包袱）

5. **handler 友好的 3 层线程模型**（见 §8.1）
   - Netty event loop（I/O）
   - 独立 muhandler 池（业务，`ThreadPoolExecutor(8, 400, 60s)`）
   - `HttpExchange.block()` 跨线程同步（保留同步写法，Netty 异步底层）

6. **可观测性**
   - `MuStats` 暴露连接数 / 请求数 / 字节数 / 状态码分布
   - 无侵入式统计

7. **可嵌入 Java 1.8 项目**
   - mu-server 用 Netty 4.1.137.Final（兼容 Java 1.8，不像 Tomcat 10 需要 Java 11+）

---

## 0.7 适用场景决策树

```
你是谁？
├── 企业 Java EE 应用，需要 Spring 全家桶 + JSP
│   └── Tomcat ✓（标准 Servlet 容器）
├── 微服务 / RESTful API + 嵌入式
│   └── mu-server / Netty ✓（轻量 + 高并发 + 易嵌入）
├── 高并发 RPC / 长连接网关 / WebSocket 服务
│   └── Netty ✓（事件驱动 + 0 拷贝）
├── 标准 Spring Boot Web
│   └── Tomcat（默认）✓ 或 Undertow / Jetty（更快）
└── 自定义二进制协议 / IoT / 嵌入式
    └── Netty ✓（最灵活）
```

---

## 0.8 总结

**Netty 本质优势**：Reactor 模式 + 异步非阻塞 + 池化 ByteBuf + 责任链 pipeline = 全链路无锁事件驱动。**代价**：开发者必须理解 event loop + 异步编程模型。

**Tomcat 本质优势**：标准化 Servlet 规范 + Spring 全家桶生态 + 同步阻塞 API（简单）。**代价**：同步模型在超高并发下吞吐受限。

**mu-server 选 Netty 的核心理由**：目标场景是"嵌入式微服务 + 高性能 RESTful + 现代 JAX-RS"，需要 Netty 的轻量 + 高并发 + 协议灵活性，避免 Servlet 容器的笨重。

---

## 相关笔记

**同目录章节**:
- [[summary|主入口]]
- [[01-architecture-overview|§1 整体架构 + 6 层架构图]]
- [[02-protocol-layer|§2 协议层]]（HTTP/1+2 + 自实现流控）
- [[03-abstraction-layer|§3 抽象层]]
- [[04-dispatch-layer|§4 分发层]]
- [[05-jax-rs|§5 JAX-RS 支持]]
- [[06-features|§6 功能特性]]
- [[07-handler-library|§7 Handler 库]]
- [[08-design-patterns|§8 关键设计模式]]
- [[09-netty-comparison|§9 Netty 对照表]]
- [[10-evolution|§10 演化对比]]
- [[11-limitations|§11 限制]]
- [[12-use-cases|§12 适用场景]]
- [[13-file-manifest|§13 关键文件清单]]

**跨版本对照**:
- [[mu-server-netty-analysis/summary|mu-server 0.0.3.6 (OMO 合成)]]
- [[mu-server-2.2.9-analysis/summary|mu-server 2.2.9 (OMO 合成)]]