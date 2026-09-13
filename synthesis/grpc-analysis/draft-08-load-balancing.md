---
title: "gRPC 负载均衡与服务发现"
category: synthesis
tags: [grpc, load-balancing, service-discovery, xds, pick-first, round-robin]
sources:
  - "gRPC Load Balancing Spec"
  - "gRPC-Java LoadBalancer 源码"
  - "xDS Service Mesh"
summary: "gRPC 负载均衡策略详解：pick_first / round_robin / xDS / grpclb + 服务发现集成"
provenance:
  extracted: 0.88
  inferred: 0.10
  ambiguous: 0.02
  base_confidence: 0.86
lifecycle: draft
lifecycle_changed: 2026-09-13
created: 2026-09-13
updated: 2026-09-13
---

# §08 gRPC 负载均衡与服务发现

## 1. 基础架构

```
ManagedChannel
   │
   ▼
NameResolver (服务发现)
   │
   ├─ DNS 解析
   ├─ 自定义注册中心 (Nacos/Consul/Eureka)
   └─ xDS (Envoy/Istio)
   │
   ▼ 返回 List<SocketAddress>
LoadBalancer (负载均衡)
   │
   ├─ pick_first (选第一个)
   ├─ round_robin (轮询)
   └─ xDS (复杂路由)
   │
   ▼ 选择 1 个 endpoint
Subchannel (HTTP/2 连接)
```

## 2. 4 种内置策略

### 2.1 pick_first（默认）

**策略**：解析所有 endpoint，连接第一个成功的，剩下的丢弃。

```java
ManagedChannel channel = ManagedChannelBuilder
    .forTarget("api-service")  // 多个 IP
    .defaultLoadBalancingPolicy("pick_first")
    .build();
```

**特点**：
- ✅ 简单，连接稳定
- ❌ 不负载均衡（只连第一个）
- ✅ 适合 endpoint 经常变化的场景（DNS 轮询）

### 2.2 round_robin

**策略**：每个 RPC 轮流选择 endpoint。

```java
ManagedChannel channel = ManagedChannelBuilder
    .forTarget("api-service")
    .defaultLoadBalancingPolicy("round_robin")
    .build();
```

**特点**：
- ✅ 真正的负载均衡
- ✅ 简单轮询
- ❌ 不感知服务器负载
- ❌ 不感知服务器健康

### 2.3 xDS（推荐，gRPC-Java 1.66+）

**xDS** = Envoy 的配置协议，gRPC-Java 完整实现。

**支持**：
- **CDS** (Cluster Discovery Service)：endpoint 列表
- **EDS** (Endpoint Discovery Service)：每个 endpoint 的负载/健康
- **LDS** (Listener Discovery Service)：listener 配置
- **RDS** (Route Discovery Service)：路由规则
- **SDS** (Secret Discovery Service）：TLS 证书

**使用**：
```java
// 1. 启用 xDS
ManagedChannel channel = ManagedChannelBuilder
    .forTarget("xds:///api-service.default.svc.cluster.local")
    .defaultLoadBalancingPolicy("xds")
    .build();

// 2. xDS 服务器地址（默认 env: GRPC_XDS_BOOTSTRAP_CONFIG）
System.setProperty("GRPC_XDS_BOOTSTRAP_CONFIG", "...");
```

**特点**：
- ✅ 全功能负载均衡 + 服务发现
- ✅ 与 Envoy/Istio 集成
- ✅ 支持流量分割、镜像、重试策略
- ⚠️ 复杂（需 xDS 服务器）

### 2.4 grpclb（已弃用）

**grpclb** 是 Google 专门的 LB 协议，**已被 xDS 取代**（gRPC-Java 1.40+ 起 deprecated）。

⚠️ **生产不要用**——所有新项目用 xDS。

## 3. NameResolver 自定义

### 3.1 DNS NameResolver（默认）

```java
// 自动使用 DNS
ManagedChannel channel = ManagedChannelBuilder
    .forAddress("api-service.example.com", 50051)
    .build();
// DNS 解析 api-service.example.com → List<InetAddress>
```

### 3.2 自定义 NameResolver

```java
public class NacosNameResolver extends NameResolver {
    private final String serviceName;
    private final NacosClient nacos;
    private Listener listener;
    
    @Override
    public void start(Listener listener) {
        this.listener = listener;
        // 1. 从 Nacos 拉取 endpoints
        nacos.subscribe(serviceName, instances -> {
            // 2. 包装为 gRPC ResolutionResult
            List<EquivalentAddressGroup> addresses = instances.stream()
                .map(i -> new EquivalentAddressGroup(
                    new InetSocketAddress(i.getIp(), i.getPort())))
                .collect(Collectors.toList());
            listener.onAddresses(addresses, Attributes.EMPTY);
        });
    }
    
    @Override
    public void refresh() {
        // 触发重新解析
    }
    
    @Override
    public void shutdown() {
        nacos.unsubscribe(serviceName);
    }
}

// 注册
NameResolverRegistry.getDefaultRegistry()
    .register(new NacosNameResolverProvider());

ManagedChannel channel = ManagedChannelBuilder
    .forTarget("nacos:///api-service")
    .build();
```

## 4. 服务发现集成实战

### 4.1 Kubernetes

```yaml
# api-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  clusterIP: None  # Headless service
  ports:
  - port: 50051
  selector:
    app: api
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: myorg/api:1.0
        ports:
        - containerPort: 50051
```

**Headless Service** 让 DNS 直接返回所有 Pod IP：

```bash
$ dig api-service.default.svc.cluster.local
api-service.default.svc.cluster.local. 30 IN A 10.0.0.1
api-service.default.svc.cluster.local. 30 IN A 10.0.0.2
api-service.default.svc.cluster.local. 30 IN A 10.0.0.3
```

**客户端代码**：
```java
ManagedChannel channel = ManagedChannelBuilder
    .forTarget("api-service.default.svc.cluster.local:50051")
    .defaultLoadBalancingPolicy("round_robin")
    .build();
```

### 4.2 xDS + Istio

```yaml
# istio sidecar 配置
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: api-service
spec:
  host: api-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: DEFAULT
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

**gRPC 客户端代码**：
```java
ManagedChannel channel = ManagedChannelBuilder
    .forTarget("xds:///api-service.default.svc.cluster.local")
    .build();
// xDS 会自动从 Istio 拉取所有配置（连接池、超时、异常检测等）
```

### 4.3 Consul

```java
public class ConsulNameResolver extends NameResolver {
    private final ConsulClient consul;
    private final String serviceName;
    
    @Override
    public void start(Listener listener) {
        consul.healthService(serviceName, healthyOnly -> {
            List<EquivalentAddressGroup> addresses = healthyOnly.stream()
                .map(s -> new InetSocketAddress(s.getAddress(), s.getPort()))
                .map(EquivalentAddressGroup::new)
                .collect(Collectors.toList());
            listener.onAddresses(addresses, Attributes.EMPTY);
        });
        
        // 长轮询
        consul.watch(serviceName, this::start);
    }
    // ...
}
```

## 5. LoadBalancer 自定义

```java
public class WeightedRoundRobinLoadBalancer extends LoadBalancer {
    private final Helper helper;
    private final Map<EquivalentAddressGroup, Integer> weights;
    
    @Override
    public void handleResolvedAddresses(
            ResolvedAddresses resolvedAddresses) {
        // 1. 提取 addresses + weights
        Map<EquivalentAddressGroup, Integer> newWeights = new HashMap<>();
        for (EquivalentAddressGroup addr : resolvedAddresses.getAddresses()) {
            int weight = resolvedAddresses.getAttributes()
                .get(WEIGHT_KEY.getKey(), addr);
            newWeights.put(addr, weight);
        }
        this.weights = newWeights;
        
        // 2. 通知 helper
        helper.refreshNameResolution();
    }
    
    @Override
    public void requestConnection() {
        // 按权重选 endpoint
        EquivalentAddressGroup selected = pickByWeight();
        helper.createSubchannel(selected, ...);
    }
    
    @Override
    public void shutdown() {
        // 清理
    }
}

// 注册
LoadBalancerRegistry.getDefaultRegistry()
    .register(new LoadBalancerProvider("weighted_round_robin") {
        @Override
        public LoadBalancer newLoadBalancer(Helper helper) {
            return new WeightedRoundRobinLoadBalancer(helper);
        }
    });
```

## 6. 负载均衡与健康检查

### 6.1 连接级健康

```java
// KeepAlive 检测连接存活
ManagedChannelBuilder
    .keepAliveTime(30, TimeUnit.SECONDS)
    .keepAliveTimeout(10, TimeUnit.SECONDS)
    .keepAliveWithoutCalls(true)
```

### 6.2 端点级健康（xDS）

xDS 通过 EDS 自动感知：
- 5xx 错误率超过阈值 → 标记 unhealthy
- outlier detection 自动剔除
- 健康后自动恢复

### 6.3 应用级健康（HealthService）

```java
// 1. 服务端暴露 HealthService
Server server = ServerBuilder.forPort(50051)
    .addService(new GreeterImpl())
    .addService(health.getHealthService())  // ←
    .build();

// 2. 业务逻辑设状态
health.setStatus("service.Greeter", ServingStatus.SERVING);

// 3. 业务不可用时
health.setStatus("service.Greeter", ServingStatus.NOT_SERVING);
```

**客户端**可以通过 HealthService 检查（需自定义 LoadBalancer）。

## 7. 实战：Kubernetes + xDS + Round Robin

```java
public class GrpcClient {
    public ManagedChannel createChannel() {
        return ManagedChannelBuilder
            .forTarget(System.getenv("SERVICE_NAME"))  // e.g. "api-service.default.svc.cluster.local"
            .defaultLoadBalancingPolicy("round_robin")  // 或 "xds" + xDS server
            .keepAliveTime(30, TimeUnit.SECONDS)
            .keepAliveTimeout(10, TimeUnit.SECONDS)
            .keepAliveWithoutCalls(true)
            .useTransportSecurity()  // mTLS in mesh
            .build();
    }
}
```

## 8. 4 种策略对比

| 策略 | 复杂度 | 负载均衡 | 服务发现 | 适用 |
|------|--------|---------|---------|------|
| **pick_first** | 低 | ❌ | DNS | 简单服务、endpoint 稳定 |
| **round_robin** | 低 | ✅ | DNS | 中小规模、无服务网格 |
| **xDS** | 高 | ✅ 高级 | ✅ 全功能 | **生产推荐** |
| **grpclb** | 中 | ✅ | ✅ | ⚠️ 已弃用 |

**生产推荐**：
- **有 Istio**：xDS
- **无服务网格**：round_robin + DNS
- **测试/开发**：pick_first

## 9. 与 OkHttp 客户端对比

| 维度 | gRPC | OkHttp |
|------|------|--------|
| **LB 策略** | 4 种内置 + 自定义 | 需自己实现 |
| **服务发现** | NameResolver 内置 | DNS / 手动 |
| **健康检查** | 内置 HealthService | 需自己实现 |
| **xDS 集成** | ✅ 原生 | ❌ |
| **Service Mesh** | Istio 一等公民 | 需手动集成 |

## 10. 常见问题

### 10.1 DNS 缓存问题

```java
// Java 默认缓存 DNS forever！需自己控制
Security.setProperty("networkaddress.cache.ttl", "30");
Security.setProperty("networkaddress.cache.negative.ttl", "10");
```

### 10.2 连接泄漏

```java
// ✅ Channel 必须 shutdown
channel.shutdown().awaitTermination(5, SECONDS);

// ❌ 忘 shutdown → 连接一直保留 → 资源泄漏
```

### 10.3 LB 策略不生效

```java
// ❌ 错：直接 forAddress
ManagedChannelBuilder.forAddress("api.com", 50051).build();
// LB 不会生效（forAddress 是单 endpoint）

// ✅ 对：forTarget + 多 IP
ManagedChannelBuilder.forTarget("api.com").build();
// forTarget 触发 NameResolver，多 IP 才走 LB
```

## 11. 设计原则

1. **生产用 xDS**——与 Istio 集成
2. **DNS 用 headless service**——返回所有 Pod IP
3. **设 keepAlive**——避免 idle 断开
4. **监控 LB 切换**——Prometheus exporter
5. **测试用 pick_first**——简单可调试

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **客户端详解**: [[draft-04-client-side]]
- **最佳实践**: [[draft-10-best-practices]]
- **综合入口**: [[summary]]