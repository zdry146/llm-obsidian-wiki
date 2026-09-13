---
title: "Kotlin 全量分析 - MOC (Map of Content)"
category: synthesis
tags: [kotlin, jvm, jetbrains, coroutines, kmp, dsl, analysis, moc, index]
sources:
  - "Kotlin 2.0+ @ JetBrains (https://kotlinlang.org/)"
  - "Kotlin in Action (Dmitry Jemerov / Svetlana Isakova)"
  - "JetBrains Blog - Kotlin Evolution"
  - "Kotlin Coroutines Guide (https://kotlinlang.org/docs/coroutines-guide.html)"
summary: "Kotlin 2.0+ 全量分析 - 语法 / 类型系统 / 函数 / 协程 / DSL / 集合 / OOP / KMP / Java 互操作 / 与 Java/Scala 对比 / 在 OkHttp/gRPC 实战中的应用"
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

# Kotlin 全量分析 - Map of Content

> 本目录是 Kotlin **2.0+** 的全量分析，涵盖语法、类型系统、函数、协程、DSL、Kotlin Multiplatform、Java 互操作，并对比 Kotlin ↔ Java ↔ Scala ↔ Groovy。最后说明 Kotlin 在 [[okhttp-analysis/summary|OkHttp]] / [[grpc-analysis/summary|gRPC]] / [[mu-server-2.4.2-analysis/summary|mu-server]] 生态中的角色。

## 主入口
- **[[summary|综合报告]]** — 执行摘要 + 生态位 + 12 章导航 + Kotlin 在 JVM 网络生态中的角色

## 12 章深度分析

| § | 子页面 | 核心内容 |
|---|---|---|
| **§00** | **[[draft-00-background\|背景与生态位]]** | JetBrains 出品、Kotlin 2.0 革命、Server Side / Android / KMP |
| §01 | [[draft-01-syntax-core\|核心语法与类型系统]] | null 安全、智能转换、data class、sealed class、object |
| §02 | [[draft-02-functions\|函数与扩展]] | 默认参数、命名参数、扩展函数、lambda、函数引用 |
| §03 | **[[draft-03-coroutines\|协程（核心）]]** | 协程基础、suspend、Structured Concurrency、Dispatcher、Flow |
| §04 | [[draft-04-dsl-builders\|DSL 与作用域函数]] | apply/with/run/also/let、scope function、Receiver 类型 |
| §05 | [[draft-05-collections\|集合与函数式]] | listOf/mapOf、Sequence vs Iterable、filter/map/reduce |
| §06 | [[draft-06-oop\|面向对象]] | sealed/data/object/enum、继承 vs 实现、扩展属性 |
| §07 | [[draft-07-kmp\|Kotlin Multiplatform]] | KMP 架构、expect/actual、iOS/Android/JVM/Linux |
| §08 | [[draft-08-java-interop\|Kotlin ↔ Java 互操作]] | 平台类型、@JvmStatic、@JvmField、@JvmOverloads |
| §09 | [[draft-09-best-practices\|最佳实践]] | 13 条铁律 + 包结构 + 命名约定 |
| §10 | [[draft-10-pitfalls\|已知坑]] | null 安全陷阱、lateinit、Java 反射、协程泄漏 |
| §11 | [[draft-11-comparison\|对比选型]] | **Kotlin vs Java vs Scala vs Groovy vs C#** |
| §12 | [[draft-12-use-cases\|应用场景]] | Android / Server / KMP / Data Science + Kotlin 在 Netty/OkHttp/gRPC 的角色 |

## 标签
`#kotlin` `#jvm` `#jetbrains` `#coroutines` `#kmp` `#dsl` `#android`

## 元信息

- **分析对象**: Kotlin 2.0+ (stable, 2024-05)
- **作者**: JetBrains（2010 立项，2016 v1.0，2024 v2.0）
- **协议**: Apache 2.0
- **目标平台**: JVM / Android / JS / Native (iOS, macOS, Linux, Windows) / Wasm
- **JDK 要求**: 编译目标 8+（最新 21+）
- **分析时间**: 2026-09-13
- **执行者**: Spark (基于 Kotlin 官方文档 + 实战经验)
- **总产出**: 1 moc + 1 summary + 12 draft ≈ 3500 行 markdown
- **关键应用**: [[okhttp-analysis/summary]] (OkHttp 4.x+ Kotlin 实现)、[[grpc-analysis/summary]] (grpc-kotlin 协程支持)

## 跨笔记链接

- **HTTP 客户端对照**: [[okhttp-analysis/summary]] — OkHttp 4.x+ 用 Kotlin 重写，5.x 协程支持
- **RPC 框架对照**: [[grpc-analysis/summary]] — grpc-kotlin 提供协程支持
- **生态三方对比**: [[draft-11-comparison]] — Kotlin vs Java vs Scala vs Groovy
- **netlib-common 模块**: 代码已用 Kotlin 编写，参考 `~/.openclaw/workspace-developer/netlib-common/`