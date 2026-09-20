---
title: "OkHttp 背景与生态位"
category: synthesis
tags: [okhttp, square, background, ecosystem, retrofit, android]
sources:
  - "OkHttp 官方文档 (https://square.github.io/okhttp/)"
  - "Square Engineering Blog"
  - "GitHub square/okhttp README"
summary: "OkHttp 的历史、定位、生态依赖图（Retrofit / Spring Cloud OpenFeign / Android 内置）"
provenance:
  extracted: 0.92
  inferred: 0.06
  ambiguous: 0.02
base_confidence: 0.90
lifecycle: draft
lifecycle_changed: 2026-09-12
created: 2026-09-12
updated: 2026-09-12
---

# §00 OkHttp 背景与生态位

## 1. 一句话定位

**OkHttp 是 Square 公司开源的 HTTP/HTTP2/HTTPS/WebSocket 客户端**，Apache 2.0 协议，**JVM 生态事实标准**——Android 自 4.x 起内置、Retrofit 强制依赖、Spring Cloud OpenFeign 默认底层。

## 2. 简史

| 时间 | 事件 |
|------|------|
| 2013-05 | Square 发布 OkHttp 1.0（解决 Android HttpURLConnection 的各种 bug） |
| 2014 | 2.x 系列：HTTP/2 支持稳定 |
| 2016 | 3.x 系列：API 重写，Interceptor 模型成熟 |
| 2019 | 4.0：用 [[kotlin-analysis/summary\|Kotlin]] 重写（保留 Java API 二进制兼容） |
| 2021 | 4.9.x：稳定主版本 |
| 2023-10 | 4.12.0：JDK 21 IPv6/IPv4 bug 修复（关键！） |
| 2024-09 | 5.0.0-alpha：[[kotlin-analysis/summary\|Kotlin]] 协程原生支持 |
| 2026 | 5.0.0 仍在 alpha，预计 GA 中 |

## 3. Square 与 OkHttp

**Square**（移动支付公司）出品，**Jake Wharton**（Android 圈大神）为主要维护者之一。Square 围绕 I/O 构建了完整生态：

```
Square I/O 生态
├── OkHttp        ← HTTP 客户端
├── Okio          ← I/O 基础库（OkHttp 强依赖）
├── Retrofit      ← 类型安全 HTTP 客户端（强依赖 OkHttp）
├── Moshi         ← JSON 解析（可选配套）
├── Picasso       ← 图片加载（可选配套）
└── LeakCanary    ← 内存泄漏检测（独立工具）
```

Square 在自家支付系统（Square POS / Cash App）大规模使用 OkHttp，**生产验证强度很高**。

## 4. 为什么 Android 内置它

Android 4.x 之前，`HttpURLConnection` 有大量 bug（DNS 缓存不刷新、连接池失效、HTTP/2 不支持、内存泄漏）。Google 评估后发现 OkHttp 最稳定，于 **Android 4.4 (KitKat, 2013)** 起将 OkHttp 作为系统内置 HTTP 客户端（`com.squareup.okhttp` 包名）。

> Android 4.4+ 的 `HttpURLConnection` 内部就是 OkHttp。

这意味着：**任何 Android 应用都已经在用 OkHttp**，只是很多人不知道。

## 5. 生态依赖图

```
┌─────────────────────────────────────────────┐
│  业务代码 (Retrofit / 直接调用 OkHttp)        │
├─────────────────────────────────────────────┤
│  Retrofit (Square 出品)                       │  ← 声明式 HTTP，强制依赖 OkHttp
├─────────────────────────────────────────────┤
│  Spring Cloud OpenFeign (Spring)              │  ← 默认底层用 OkHttp 替代 HttpURLConnection
├─────────────────────────────────────────────┤
│  OkHttp  ← 你读的这个                         │
├─────────────────────────────────────────────┤
│  Okio (Square 自研 I/O 库)                    │  ← Buffer / Source / Sink
├─────────────────────────────────────────────┤
│  JDK NIO / Conscrypt (TLS) / Android API      │
└─────────────────────────────────────────────┘
```

## 6. 商业采用

| 公司 | 用法 |
|------|------|
| **Square**（自身） | 支付 API 客户端 |
| **Twitter** | 移动端 API |
| **Pinterest** | 后端微服务调用 |
| **Uber** | 移动端 + 后端 |
| **Slack** | Web 客户端 |
| **Tinder** | 移动端 |
| **京东 / 美团 / 字节跳动** | 国内大厂广泛使用 |

**金融场景**也有采用（Square 自家），可靠性有保障。

## 7. 协议与许可

| 维度 | 数据 |
|------|------|
| 协议 | Apache License 2.0 |
| 商用 | ✅ 免费，可商用，无需公开修改 |
| 修改 | ✅ 可修改 |
| 商标 | 不能用 Square 商标推广衍生品 |

**对比 Apache HttpClient**（也是 Apache 2.0）：OkHttp 更小、API 更现代。

## 8. 关键事实速记

- **GitHub**: https://github.com/square/okhttp
- **官方文档**: https://square.github.io/okhttp/
- **最新 stable**: 4.12.0（2023-10）
- **最新 alpha**: 5.0.0-alpha.14（2024-09，Kotlin 协程支持）
- **依赖**: Okio 3.x + [[kotlin-analysis/summary\|Kotlin]] stdlib（4.x 起）
- **JDK**: Java 8+（4.x/5.x），Android API 21+（5.0 Lollipop）

## 9. 设计哲学（Square 官方表述）

Square 在多篇工程博客中反复强调 OkHttp 的设计原则：

1. **拦截器链作为一等公民** —— 所有横切关注点（日志、鉴权、重试、缓存）都通过 Interceptor 注入
2. **Okio 抽象替代 NIO** —— 让 I/O 代码像写 String 一样简单
3. **连接池透明** —— 用户不需要管理 TCP 连接生命周期
4. **HTTP/2 默认开启** —— 多路复用透明生效
5. **失败重试透明** —— 网络抖动自动重连，遵循 RFC 7231

> "我们写 OkHttp 是因为 Java 自带的 HTTP 客户端糟透了，而 Android 的更糟。"
> —— Jesse Wilson（OkHttp 主要作者），GOTO 2015 演讲

## 10. 与 mu-server 的对称性

| 维度 | OkHttp | [[mu-server-2.4.2-analysis/summary\|mu-server]] |
|------|--------|-----------|
| 角色 | HTTP **客户端**（出站） | HTTP **服务端**（入站） |
| 底层 | Okio | Netty |
| 用户接口 | 同步 + 异步 + 协程 | 同步 + 异步（block 模式） |
| 拦截器链 | ✅ 6 层 | ✅ handler chain |
| Square 出品 | ✅ | ❌（3redronin 个人项目） |
| Apache 2.0 | ✅ | ✅ |

两个都是 Square/Netty 生态的产物，**OkHttp 是"出站"，mu-server 是"入站"**，构成完整 HTTP 双向栈。详见 [[summary]] §6。

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **拦截器链**: [[draft-02-interceptors]]
- **实战踩坑**: [[draft-10-known-issues]]
- **综合入口**: [[summary]]
