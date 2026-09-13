---
title: "gRPC 全量分析 - MOC (Map of Content)"
category: synthesis
tags: [grpc, protobuf, rpc, http2, netty, cncf, analysis, moc, index]
sources:
  - "gRPC 1.66+ @ grpc/grpc-java (https://github.com/grpc/grpc-java)"
  - "gRPC Core Spec (https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)"
  - "Protocol Buffers v3 (https://protobuf.dev/)"
  - "CNCF gRPC (https://www.cncf.io/projects/grpc/)"
summary: "gRPC 全量分析 - 架构 / 协议 / 服务定义 / 客户端 / 服务端 / 4 种流式 / 拦截器 / 负载均衡 / 错误处理 / 与 Netty 和 mu-server 对比"
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

# gRPC 全量分析 - Map of Content

> 本目录是 gRPC **1.66+ (Java)** 的源码级全量分析，附带 .proto 代码生成、4 种流式模式、双向拦截器、与 [[mu-server-2.4.2-analysis/summary|mu-server]] 和 Netty 的完整对比。

## 主入口
- **[[summary|综合报告]]** — 执行摘要 + 生态位 + 13 章导航 + gRPC vs Netty vs mu-server 对比

## 13 章深度分析

| § | 子页面 | 核心内容 |
|---|---|---|
| **§00** | **[[draft-00-background\|背景与生态位]]** | Google 出品、CNCF 毕业、protobuf 序列化、跨语言 |
| §01 | [[draft-01-architecture\|核心架构]] | Channel / Stub / Service / Server 四件套 + 调用时序 |
| §02 | [[draft-02-wire-protocol\|线协议]] | HTTP/2 帧 + protobuf wire format + Length-Prefixed Message |
| §03 | [[draft-03-service-definition\|服务定义与代码生成]] | .proto 文件 + protoc + Maven/Gradle 插件 |
| §04 | [[draft-04-client-side\|客户端详解]] | ManagedChannel / Stub / CallOptions / 异步 Future |
| §05 | [[draft-05-server-side\|服务端详解]] | ServerBuilder / Service / ServerServiceDefinition |
| §06 | [[draft-06-streaming\|4 种流式模式]] | Unary / Server-streaming / Client-streaming / Bidirectional |
| §07 | [[draft-07-interceptors\|拦截器]] | ClientInterceptor + ServerInterceptor 完整示例 |
| §08 | [[draft-08-load-balancing\|负载均衡与服务发现]] | pick_first / round_robin / xDS / grpclb |
| §09 | [[draft-09-error-handling\|错误处理与状态码]] | Status / StatusException / StatusRuntimeException / 16 个 code |
| §10 | [[draft-10-best-practices\|最佳实践]] | Channel 复用 / 上下文传播 / 超时 / 压缩 |
| §11 | [[draft-11-known-issues\|已知坑]] | HTTP/2 连接泄漏 / 阻塞线程 / 序列化兼容性 |
| §12 | [[draft-12-comparison\|对比选型]] | **gRPC vs Netty vs mu-server vs REST vs GraphQL** |

## 标签
`#grpc` `#protobuf` `#rpc` `#http2` `#netty` `#cncf` `#java`

## 元信息
- **分析对象**: gRPC-Java 1.66+ (grpc-java / grpc-core / grpc-protobuf / grpc-stub / grpc-netty)
- **协议**: Apache 2.0
- **核心依赖**: Netty 4.1.x + protobuf-java 3.x + Guava
- **JDK 要求**: Java 8+ (1.66+), Android API 21+
- **分析时间**: 2026-09-13
- **执行者**: Spark (直接读 grpc/grpc-java 源码 + CNCF 文档)
- **总产出**: 13 draft + 1 moc + 1 summary ≈ 3500 行 markdown
- **关键对比**: [[draft-12-comparison]] — gRPC / Netty / mu-server 三方生态对照

## 跨笔记链接
- **HTTP 客户端对照**: [[okhttp-analysis/summary]] — OkHttp 是单次 HTTP 调用，gRPC 是 RPC 框架
- **Netty 服务端对照**: [[mu-server-2.4.2-analysis/summary]] — mu-server 是 gRPC server 的 HTTP 对等物
- **Netty 基础**: Netty 是 gRPC 和 mu-server 的共同基础