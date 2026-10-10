---
title: "Java 全量分析综合报告 — 主入口"
category: synthesis
tags: [java, jvm, openjdk, lts, lambda, stream, virtual-thread, sealed, records, pattern-matching, reactor, analysis, index]
sources:
  - "OpenJDK JDK 25 Feature List (https://openjdk.org/projects/jdk/25/)"
  - "JEP Index (https://openjdk.org/jeps/0)"
  - "Java Language Specification (JLS)"
  - "Java 25 — InfoQ (Sep 2025)"
  - "Foojay JDK 25 Almanac"
summary: "Java 8 → 25 全量分析 — 7 个 chapter：基础(Lambda/Stream/Optional)→ 现代(模块/HTTP/var)→ 数据+模式(records/sealed/pattern)→ 虚拟线程(Java 21 LTS)→ 现代并发(22-25)→ 版本速查→ Reactor 响应式"
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

# Java 全量分析综合报告 — 主入口

> **版本覆盖**: Java **8** (2014, LTS) → Java **25** (2025-09-16, LTS)
> **作者**: OpenJDK / Oracle
> **协议**: GPLv2 + Classpath Exception (OpenJDK)
> **目标平台**: JVM (主流)，GraalVM Native
> **最新 LTS**: Java 25 (released 2025-09-16, EOL 2030-09)
> **分析时间**: 2026-10-03
> **执行者**: Spark

## TL;DR

Java 从 Java 8 (2014) 到 Java 25 (2025) 的 11 年里完成了**从"面向对象 + 样板"到"面向数据 + 函数式 + 虚拟线程"的根本范式转移**。最关键的 3 个里程碑：

1. **Java 8 (2014, LTS)** — 引入 lambda + Stream + Optional + `java.time`，Java 第一次有了现代函数式能力
2. **Java 17 (2021, LTS)** — records (final) + sealed classes (final) + pattern matching `instanceof` (final)，Java 第一次有了"数据载体"一等公民
3. **Java 21 (2023, LTS)** — virtual threads (final) + pattern switch (final) + record patterns (final)，Java 第一次有了"百万级并发"原生支持
4. **Java 25 (2025, LTS)** — Module Import Declarations + Compact Source Files + Scoped Values (final)，Java 第一次有了"现代化教学/脚本"体验 + 现代化上下文传递

## 7 章导航

| # | 章节 | 版本 | 主题 |
|---|---|---|---|
| 1 | [[draft-01-java-8-foundations]] | **Java 8** | Lambda / Stream / Optional / CompletableFuture / `java.time` / default methods |
| 2 | [[draft-02-java-9-11-modernization]] | Java 9 → 11 | 模块系统 / 集合工厂方法 / `var` / HTTP Client / 单文件启动 |
| 3 | [[draft-03-java-14-17-data-and-patterns]] | Java 14 → 17 | records / sealed / pattern matching / switch 表达式 / text blocks |
| 4 | [[draft-04-java-21-lts-virtual-threads]] | **Java 21 LTS** | virtual threads / sequenced collections / pattern switch / record patterns / string templates |
| 5 | [[draft-05-java-22-25-modern-concurrency]] | Java 22 → 25 | structured concurrency / scoped values / FFM API / primitive patterns / module imports / compact source files |
| 6 | [[draft-06-version-timeline-cheatsheet]] | Java 8 → 25 | 版本时间线 + 速查 + 升级路径 |
| 7 | [[draft-07-reactor]] | 跨版本 | **Project Reactor** — Mono / Flux / 操作符 / 调度 / 背压 / Spring WebFlux / R2DBC |

## 版本生态位

### LTS 节奏

```
Java 8  (2014) ─┐
                ├─ 6 年跨度
Java 11 (2018) ─┤
                ├─ 3 年
Java 17 (2021) ─┤
                ├─ 3 年
Java 21 (2023) ─┤
                ├─ 2 年
Java 25 (2025) ─┘  ← 当前最新 LTS
```

### 各版本的"标志性特征"

| 版本 | 主题 | 一句话 |
|---|---|---|
| Java 8 | 函数式 | "Java 终于有了 lambda" |
| Java 9 | 模块化 | "应用可以分模块了"（但生态迟迟不跟） |
| Java 10 | 类型推断 | "局部 `var`，但不能用在字段" |
| Java 11 | HTTP + 工具 | "内置 HTTP Client，告别第三方" |
| Java 14-16 | 数据 | "records：POJO 一行写完" |
| Java 17 | 封闭 + 模式 | "sealed + pattern matching，类型驱动" |
| Java 21 | 虚拟线程 | "百万级并发原生支持" |
| Java 25 | 现代化 + 性能 | "compact source files + AOT + compact headers" |

## 核心趋势线

### 趋势 1：样板代码衰减

```java
// Java 7 之前
public class User {
    private final String name;
    private final int age;
    public User(String name, int age) { this.name = name; this.age = age; }
    public String getName() { return name; }
    public int getAge() { return age; }
    public boolean equals(Object o) { ... }
    public int hashCode() { ... }
    public String toString() { ... }
}

// Java 8
public class User {
    private final String name;
    private final int age;
    public User(String name, int age) { this.name = name; this.age = age; }
    // ... 还是 60 行
}

// Java 16+ records
public record User(String name, int age) {}
// 一行，自动 equals/hashCode/toString
```

### 趋势 2：并发模型革新

```
Java 8 (2014):   CompletableFuture + ExecutorService     ← 基于线程池的异步
Java 19 (2022):  Virtual Threads (preview 1)            ← 协程式用户态线程
Java 21 (2023):  Virtual Threads (final) + Structured Concurrency (incubator)
Java 25 (2025):  Scoped Values (final) + Structured Concurrency (5th preview)
```

### 趋势 3：类型驱动控制流

```java
// Java 16 之前
if (obj instanceof String) {
    String s = (String) obj;        // 强制转型
    System.out.println(s.length());
}

// Java 16+
if (obj instanceof String s) {       // pattern binding
    System.out.println(s.length());
}

// Java 21+ record patterns
if (obj instanceof User(String name, int age)) {   // 解构
    System.out.println(name + " is " + age);
}
```

### 趋势 4：模块化教学 + 学习曲线降低

```java
// Java 25 compact source files
// 文件名可以是 Hello.java
import module java.base;

void main() {
    IO.println("Hello, Java 25!");
}
// 没有 class、没有 public static void main、不写 import java.util.*
// 学生第一个程序从 8 行变 4 行
```

## 实战应用

Java 在这些生态中的角色（cross-references）：

- **网络客户端**: [[okhttp-analysis/summary]] — OkHttp 5+ 全面适配 Java 21+ virtual threads
- **RPC 框架**: [[grpc-analysis/summary]] — grpc-java 已用 virtual threads 重写 server 端
- **HTTP 服务器**: [[mu-server-2.4.2-analysis/summary]] — 内部使用虚拟线程 + Scoped Values
- **与 Kotlin 对比**: [[kotlin-analysis/summary]] — Kotlin 一直领先，但 Java 25 在 pattern matching 上一举追上

## 反模式警告

❌ **盲目用 Java 25 所有预览特性** — 预览 API 不保证稳定，每次升级可能破坏
❌ **从 Java 8 直接跳 Java 25 跳过中间版本** — `sealed`、`var`、record patterns 的语义微妙不同
❌ **virtual threads 替代线程池而不调优** — pinning 问题在 synchronized 块还存在（Java 24 才解决）
❌ **用 `var` 偷懒省略类型** — 公开 API 仍然要明确类型，可读性优先
❌ **单文件启动用在生产** — JEP 463 是为教学/脚本设计的，不是替代 `java -jar`

## 关键设计原则（沿用至今）

1. **向后兼容优先** — Java 25 仍然能跑 Java 5 的代码
2. **预览机制** — 新特性先 preview 1-2 个版本，再 final，给生态留时间
3. **LTS 节奏** — 每 2-3 年一个 LTS，企业有升级窗口
4. **JEP 流程** — 所有语言特性走 JEP（Java Enhancement Proposal），公开讨论
5. **OpenJDK 治理** — 2017 起 Oracle + 社区共同演进，多 vendor 可发布构建

## 相关笔记

- **Map of Content**: [[moc]]
- **TypeScript**: [[../typescript-analysis/summary]]
- **Kotlin**: [[../kotlin-analysis/summary]]
- **三方对照**: [[../java-kotlin-typescript-comparison]] — Java vs Kotlin vs TypeScript 全方位对比
- **综合入口**: [[../summary]]
