---
title: "Java 全量分析 - MOC (Map of Content)"
category: synthesis
tags: [java, jvm, openjdk, lts, analysis, moc, index]
sources:
  - "OpenJDK JDK 25 Feature List"
  - "JEP Index (https://openjdk.org/jeps/0)"
  - "Java 25 — InfoQ (Sep 2025)"
summary: "Java 8 → 25 全量分析 - 语言特性演进、并发模型革新、模块化、模式匹配、虚拟线程、Project Reactor 响应式"
provenance:
  extracted: 0.85
  inferred: 0.12
  ambiguous: 0.03
base_confidence: 0.85
lifecycle: draft
lifecycle_changed: 2026-10-03
created: 2026-10-03
updated: 2026-10-03
---

# Java 全量分析 - Map of Content

> 本目录覆盖 Java **8 → 25** 的语言特性、并发模型、数据载体、模式匹配、虚拟线程、模块化等核心演进。Java 25 是 2025-09-16 发布的最新 LTS。

## 主入口
- **[[summary|综合报告]]** — 执行摘要 + 6 章导航 + 4 大趋势线 + 生态位

## 7 章深度分析

| # | 章节 | 覆盖版本 | 主要 JEP |
|---|---|---|---|
| 1 | [[draft-01-java-8-foundations]] | Java 8 (2014) | Lambda / Stream / Optional / CompletableFuture / `java.time` / default methods |
| 2 | [[draft-02-java-9-11-modernization]] | Java 9-11 | Jigsaw module / List.of / `var` / HTTP Client |
| 3 | [[draft-03-java-14-17-data-and-patterns]] | Java 14-17 | records / sealed / pattern matching / switch expr / text blocks |
| 4 | [[draft-04-java-21-lts-virtual-threads]] | Java 21 (LTS) | virtual threads / sequenced collections / record patterns / unnamed |
| 5 | [[draft-05-java-22-25-modern-concurrency]] | Java 22-25 | structured concurrency / scoped values / FFM / module imports / compact source files |
| 6 | [[draft-06-version-timeline-cheatsheet]] | Java 8-25 | 版本时间线 + 速查表 + 升级路径 |
| 7 | [[draft-07-reactor]] | 跨版本 | Project Reactor — Mono / Flux / 操作符 / 调度 / 背压 / WebFlux / R2DBC |

## 按主题跳转

### 函数式编程
- Java 8 Lambda + Stream — [[draft-01-java-8-foundations#1-lambda-表达式]]
- Java 8 Optional — [[draft-01-java-8-foundations#3-optional]]
- Java 21+ 函数式风格 — [[draft-04-java-21-lts-virtual-threads#3-pattern-matching-switch-standard]]

### 数据载体
- Java 16 records — [[draft-03-java-14-17-data-and-patterns#1-records]]
- Java 17 sealed classes — [[draft-03-java-14-17-data-and-patterns#2-sealed-classes]]
- Java 21 record patterns — [[draft-04-java-21-lts-virtual-threads#4-record-patterns-standard]]

### 并发模型
- Java 8 CompletableFuture — [[draft-01-java-8-foundations#4-completablefuture]]
- Java 9 Reactive Streams Flow — [[draft-02-java-9-11-modernization#4-reactive-streams-flow-api]]
- Java 21 virtual threads — [[draft-04-java-21-lts-virtual-threads#1-virtual-threads-standard]]
- Java 25 structured concurrency — [[draft-05-java-22-25-modern-concurrency#9-structured-concurrency-preview]]
- Java 25 scoped values — [[draft-05-java-22-25-modern-concurrency#11-scoped-values-final]]

### 模块化
- Java 9 JPMS — [[draft-02-java-9-11-modernization#1-模块系统-jigsaw]]
- Java 25 Module Import Declarations — [[draft-05-java-22-25-modern-concurrency#6-module-import-declarations-final]]

### 模式匹配演进
- Java 16 instanceof pattern — [[draft-03-java-14-17-data-and-patterns#3-pattern-matching-instanceof]]
- Java 14 switch expressions — [[draft-03-java-14-17-data-and-patterns#4-switch-表达式]]
- Java 21 pattern switch — [[draft-04-java-21-lts-virtual-threads#3-pattern-matching-switch-standard]]
- Java 25 primitive patterns — [[draft-05-java-22-25-modern-concurrency#5-primitive-patterns-preview]]

### 工具/IO
- Java 11 HTTP Client — [[draft-02-java-9-11-modernization#6-http-client]]
- Java 11 单文件启动 — [[draft-02-java-9-11-modernization#7-单文件启动]]
- Java 25 Compact Source Files — [[draft-05-java-22-25-modern-concurrency#7-compact-source-files-final]]

### 升级路径
- 何时升 Java 25 — [[draft-06-version-timeline-cheatsheet#5-升级路径建议]]
- LTS 节奏 — [[summary#lts-节奏]]

## 相关笔记

- **TypeScript**: [[../typescript-analysis/summary]]
- **Kotlin**: [[../kotlin-analysis/summary]]
- **综合入口**: [[../summary]]
