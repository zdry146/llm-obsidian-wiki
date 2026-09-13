---
title: "gRPC 服务端详解 (ServerBuilder / BindableService)"
category: synthesis
tags: [grpc, server, serverbuilder, bindableservice, serverinterceptor]
sources:
  - "gRPC-Java NettyServer 源码"
  - "ServerBuilder 源码"
  - "gRPC User Guide - Server"
summary: "gRPC 服务端：ServerBuilder + BindableService + 拦截器链 + 优雅关闭 + 与 Netty/mu-server 对比"
provenance:
  extracted: 0.90
  inferred: 0.08
  ambiguous: 0.02
  base_confidence: 0.88
lifecycle: draft
lifecycle_changed: 2026-09-13
created: 2026-09-13
updated: 2026-09-13
---

# §05 gRPC 服务端详解（ServerBuilder / BindableService）

## 1. 服务端核心组件

```
应用
 ├─ Server（监听 TCP，可注册多个 Service）
 │   ├─ ServerBuilder.forPort(port)
 │   ├─ addService(BindableService)
 │   └─ 底层 Netty ServerBootstrap
 │
 ├─ BindableService（业务实现）
 │   ├─ 继承 .proto 生成的 MyServiceImplBase
 │   ├─ 实现各方法
 │   └─ 实现 serviceDescriptor()
 │
 ├─ ServerCall（单次调用，gRPC 内部）
 │   ├─ ServerCallImpl 实现
 │   ├─ sendMessage() / close()
 │   └─ request(N) / halfClose()
 │
 ├─ ServerInterceptor（服务端拦截器）
 │   ├─ 鉴权、监控、日志
 │   └─ 调用链：InterceptorContext → Service
 │
 └─ ServerCallHandler（实际处理业务）
     └─ 默认 UnaryServerCallHandler
```

## 2. ServerBuilder 完整配置

```java
Server server = ServerBuilder.forPort(50051)
    // 协议
    .useTransportSecurity()
    
    // 服务注册
    .addService(new GreeterImpl())
    .addService(new HealthServiceImpl())  // 可注册多个
    
    // 拦截器（按顺序执行）
    .intercept(new AuthServerInterceptor())
    .intercept(new MetricsServerInterceptor())
    .intercept(new LoggingServerInterceptor())
    
    // Fallback（处理未知 service）
    .fallbackHandlerRegistry(new MyFallbackRegistry())
    
    // 压缩
    .compressorRegistry(CompressorRegistry.getDefaultInstance())
    .decompressorRegistry(DecompressorRegistry.getDefaultInstance())
    
    // 性能
    .keepAliveTime(30, TimeUnit.SECONDS)
    .keepAliveTimeout(10, TimeUnit.SECONDS)
    .permitKeepAliveWithoutCalls(false)
    .permitKeepAliveTime(20, TimeUnit.SECONDS)
    .maxConnectionIdle(60, TimeUnit.SECONDS)
    .maxConcurrentCallsPerConnection(100)
    
    // 用户代理
    .userAgent("my-server/1.0")
    
    .build();
```

### 2.1 关键配置详解

| 选项 | 默认 | 作用 |
|------|------|------|
| `keepAliveTime` | 2h | 服务端多久发 PING |
| `permitKeepAliveWithoutCalls` | false | 是否允许客户端无 RPC 时发 PING |
| `permitKeepAliveTime` | 20s | 客户端 PING 最小间隔 |
| `maxConnectionIdle` | 无 | 连接空闲多久关闭 |
| `maxConcurrentCallsPerConnection` | 无上限 | 单连接最多并发 RPC |
| `maxInboundMessageSize` | 4 MB | 请求超过此值断流 |
| `maxInboundMetadataSize` | 8 KB | metadata 超过此值断流 |

**生产推荐**：
```java
.keepAliveTime(60, TimeUnit.SECONDS)
.permitKeepAliveWithoutCalls(true)
.permitKeepAliveTime(20, TimeUnit.SECONDS)
.maxConnectionIdle(5, TimeUnit.MINUTES)
.maxConcurrentCallsPerConnection(200)
.maxInboundMessageSize(64 * 1024 * 1024)
```

## 3. BindableService 实现模板

```java
// 1. 继承 .proto 生成的基类
public class GreeterImpl extends GreeterGrpc.GreeterImplBase {
    
    // 2. 实现 Unary 方法
    @Override
    public void sayHello(HelloRequest req, StreamObserver<HelloReply> responseObserver) {
        HelloReply reply = HelloReply.newBuilder()
            .setMessage("Hello, " + req.getName())
            .build();
        responseObserver.onNext(reply);
        responseObserver.onCompleted();
    }
    
    // 3. 实现 Server streaming
    @Override
    public void lotsOfReplies(HelloRequest req, StreamObserver<HelloReply> responseObserver) {
        for (int i = 0; i < 10; i++) {
            HelloReply reply = HelloReply.newBuilder()
                .setMessage("Reply " + i)
                .build();
            responseObserver.onNext(reply);
        }
        responseObserver.onCompleted();
    }
    
    // 4. 实现 Client streaming
    @Override
    public StreamObserver<HelloRequest> lotsOfGreetings(
            StreamObserver<HelloReply> responseObserver) {
        return new StreamObserver<HelloRequest>() {
            private final List<String> names = new ArrayList<>();
            
            @Override public void onNext(HelloRequest req) {
                names.add(req.getName());
            }
            @Override public void onError(Throwable t) { /* ... */ }
            @Override public void onCompleted() {
                String greeting = "Hello, " + String.join(", ", names);
                responseObserver.onNext(HelloReply.newBuilder()
                    .setMessage(greeting).build());
                responseObserver.onCompleted();
            }
        };
    }
}
```

**关键点**：
- Unary: 返回 `void`，发响应通过 `StreamObserver.onNext()`
- Server streaming: 返回 `void`，多次 `onNext()`，最后 `onCompleted()`
- Client streaming: 返回 `StreamObserver<Request>`，客户端多次 `onNext()`，最终 `onCompleted()`
- Bidirectional: 返回 `StreamObserver<Request>`，双向交互

## 4. 优雅关闭（生产必懂）

```java
Server server = ServerBuilder.forPort(50051)
    .addService(new GreeterImpl())
    .build();

server.start();
System.out.println("Server started on " + server.getPort());

// 1. 注册 shutdown hook（SIGTERM/SIGINT 触发）
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    System.out.println("Received shutdown signal");
    
    // 2. 优雅 shutdown（拒绝新 RPC，等待 in-flight 完成）
    server.shutdown();
    try {
        // 3. 等待 in-flight 完成（最多 30s）
        if (!server.awaitTermination(30, TimeUnit.SECONDS)) {
            System.out.println("Forcing shutdown");
            server.shutdownNow();  // 强制
        }
    } catch (InterruptedException e) {
        server.shutdownNow();
        Thread.currentThread().interrupt();
    }
}));

// 4. 主线程阻塞
server.awaitTermination();
```

### 4.1 优雅关闭时序

```
T+0s    SIGTERM received
        │
        ▼
server.shutdown()
   ├─ 拒绝新 RPC（返回 UNAVAILABLE）
   └─ 等待 in-flight RPC 完成
        │
T+0~30s in-flight RPC 完成
        │
        ▼
server.awaitTermination(30s) 返回 true
        │
        ▼
Netty 关闭 → 释放端口 → 进程退出
```

**关键配置**：
- `maxConnectionIdle`：空闲连接立即关闭
- `maxConcurrentCallsPerConnection`：限制并发，便于快速完成

## 5. 服务端拦截器

```java
public class AuthServerInterceptor implements ServerInterceptor {
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {
        
        // 1. 验证 metadata
        String auth = headers.get(AUTH_KEY);
        if (!isValid(auth)) {
            // 2. 拒绝调用
            call.close(Status.UNAUTHENTICATED.withDescription("invalid token"), 
                       new Metadata());
            return new ServerCall.Listener<ReqT>() {};  // 空 listener，不处理请求
        }
        
        // 3. 放行
        return next.startCall(call, headers);
    }
}

// 注册
Server server = ServerBuilder.forPort(50051)
    .addService(new GreeterImpl())
    .intercept(new AuthServerInterceptor())  // 第一个拦截器（最外层）
    .intercept(new MetricsServerInterceptor())
    .build();
```

**拦截器链顺序**（注册顺序）：
```
Client Request
   │
   ▼
AuthServerInterceptor  ← 第 1 个（最外层，先执行）
   │
   ▼
MetricsServerInterceptor  ← 第 2 个
   │
   ▼
ServerCallHandler  ← 业务处理
   │
   ▼
Service.method  ← 实际方法
```

详见 [[draft-07-interceptors]]。

## 6. 完整生产级服务端示例

```java
public class GreeterServer {
    private static final Logger logger = LoggerFactory.getLogger(GreeterServer.class);
    
    public static void main(String[] args) throws IOException, InterruptedException {
        // 1. 业务实现
        Server server = ServerBuilder.forPort(50051)
            .addService(new GreeterImpl())
            .addService(ProtoReflectionService.newInstance())  // 服务反射（grpcurl 需要）
            
            // 拦截器
            .intercept(new AuthServerInterceptor())
            .intercept(new LoggingServerInterceptor(logger))
            .intercept(new MetricsServerInterceptor(Metrics.globalRegistry))
            .intercept(new TracingServerInterceptor(OpenTelemetry.get()))
            
            // 性能
            .keepAliveTime(60, TimeUnit.SECONDS)
            .permitKeepAliveWithoutCalls(true)
            .maxConnectionIdle(5, TimeUnit.MINUTES)
            .maxConcurrentCallsPerConnection(200)
            .maxInboundMessageSize(64 * 1024 * 1024)
            
            // TLS
            .useTransportSecurity(
                Certs.loadCertChain("server.crt"),
                Certs.loadPrivateKey("server.key"))
            
            .build();
        
        // 2. 启动
        server.start();
        logger.info("Server started on {}", server.getPort());
        
        // 3. Shutdown hook
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            logger.info("Shutting down gRPC server");
            try {
                server.shutdown().awaitTermination(30, TimeUnit.SECONDS);
            } catch (InterruptedException e) {
                server.shutdownNow();
                Thread.currentThread().interrupt();
            }
        }));
        
        // 4. 阻塞
        server.awaitTermination();
    }
}
```

## 7. Service 注册进阶

### 7.1 多个 Service

```java
Server server = ServerBuilder.forPort(50051)
    .addService(new GreeterImpl())
    .addService(new UserServiceImpl())
    .addService(new OrderServiceImpl())
    .addService(HealthStatusManager.getInstance()  // 标准健康检查服务
        .getHealthService())
    .addService(ProtoReflectionService.newInstance())  // 服务反射
    .build();
```

### 7.2 ProtoReflectionService（推荐）

```java
import io.grpc.protobuf.services.ProtoReflectionService;

Server server = ServerBuilder.forPort(50051)
    .addService(new GreeterImpl())
    .addService(ProtoReflectionService.newInstance())  // ← 加这个
    .build();
```

**作用**：
- 让 `grpcurl` 能列出所有服务
- 让 gRPC UI 工具能发现服务
- **生产建议**：开启（grpc-java 1.60+ 默认）

### 7.3 健康检查

```java
HealthStatusManager health = new HealthStatusManager();

Server server = ServerBuilder.forPort(50051)
    .addService(new GreeterImpl())
    .addService(health.getHealthService())
    .build();

// 设置服务健康状态
health.setStatus(HelloRequest.getDescriptor().getFullName(), 
                  ServingStatus.SERVING);
```

**作用**：Kubernetes liveness/readiness probe 标准服务。

## 8. Netty Server 底层（gRPC 是怎么构建的）

```java
// gRPC 内部用 Netty 实现
public final class NettyServer extends Server {
    private final ServerBootstrap bootstrap;
    private final NioEventLoopGroup bossGroup;
    private final NioEventLoopGroup workerGroup;
    
    // start() 内部
    public NettyServer start() {
        bossGroup = new NioEventLoopGroup(1);    // boss：接受连接
        workerGroup = new NioEventLoopGroup(0);  // worker：I/O
        
        bootstrap = new ServerBootstrap()
            .group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .childHandler(new NettyServerHandler(...));  // gRPC 核心 handler
        
        // bind
        ChannelFuture cf = bootstrap.bind(port).sync();
        return this;
    }
}
```

**关键洞察**：**gRPC 服务端是 Netty 之上的应用**——和 [[mu-server-2.4.2-analysis/summary|mu-server]] 一样。

## 9. gRPC vs [[mu-server-2.4.2-analysis/summary\|mu-server]] 服务端对比

| 维度 | gRPC Server | [[mu-server-2.4.2-analysis/summary\|mu-server]] |
|------|-------------|-----------|
| **作者** | Google | 3redronin |
| **协议** | HTTP/2 + Protobuf（强制） | HTTP/1.1 + HTTP/2（任意序列化） |
| **API 风格** | 继承 .proto 基类 | Handler / Routes |
| **同步阻塞** | Stub 异步 + 业务线程池 | `HttpExchange.block()` |
| **流式** | 4 种协议级流式 | SSE only |
| **类型安全** | ✅ 编译期 | ❌ 运行时 |
| **跨语言** | ✅ 11+ 语言 | ❌ Java only |
| **性能** | 高（HTTP/2 多路复用） | 高（Netty + HttpExchange.block） |
| **生态** | K8s/etcd/Istio | 嵌入式服务 |
| **学习曲线** | 陡（.proto + 流式 + interceptor） | 中（Handler chain） |
| **适用** | 微服务 RPC、跨语言服务 | 嵌入式 HTTP 服务、单体 Java 应用 |

**关键洞察**：
- **gRPC 服务端** 适合**微服务内部通信**（强类型 + 跨语言 + 流式）
- **mu-server** 适合**嵌入式 HTTP 服务**（轻量、Servlet 风格、Java only）

## 10. 与 Netty 关系（一图）

```
┌────────────────────────────────────────┐
│ 应用代码                                │
├────────────────────────────────────────┤
│ gRPC Server（Handler 抽象）              │
├────────────────────────────────────────┤
│ NettyServerHandler（gRPC 内部）          │
├────────────────────────────────────────┤
│ Netty ServerBootstrap / EventLoop        │
├────────────────────────────────────────┤
│ Java NIO                                │
└────────────────────────────────────────┘
```

**Netty 抽象层级**：
```
Level 0: Java NIO (Selector, SocketChannel)
Level 1: Netty (ChannelHandler, EventLoop, ByteBuf)  ← gRPC 和 mu-server 在此
Level 2: gRPC Server / mu-server / OkHttp           ← 本报告对象
Level 3: 应用代码
```

## 11. 关键设计原则

1. **BindableService 继承 .proto 基类**——避免自己写
2. **拦截器管横切关注点**——鉴权、日志、监控、追踪
3. **Shutdown hook 必加**——K8s SIGTERM 友好
4. **maxInboundMessageSize 必设**——防 DoS
5. **keepAlive 配置按场景调**——内网可短，公网要长

## 12. 反模式

❌ **业务逻辑在 ServerInterceptor 里**——应该用 Interceptor 做横切，业务在 Service
❌ **shutdown() 后才 awaitTermination**——应该先 shutdown 再等
❌ **忘注册 ProtoReflectionService**——grpcurl 无法工作
❌ **maxInboundMessageSize 不设**——4MB 限制容易被攻击
❌ **Server streaming 不响应 onError**——客户端会一直等

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **客户端详解**: [[draft-04-client-side]]
- **拦截器**: [[draft-07-interceptors]]
- **HTTP 服务端对照**: [[mu-server-2.4.2-analysis/summary]]
- **综合入口**: [[summary]]