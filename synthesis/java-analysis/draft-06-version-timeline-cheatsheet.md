---
title: "Java 8-25 版本时间线 + 速查表"
category: synthesis
tags: [java, version-timeline, lts, cheatsheet, upgrade-path]
sources:
  - "OpenJDK JDK Release Notes"
  - "Oracle Java SE Support Roadmap"
  - "JEP Index"
summary: "Java 8 → 25 版本时间线 + LTS 节奏 + 各版本关键 JEP + 升级路径建议"
provenance:
  extracted: 0.92
  inferred: 0.06
  ambiguous: 0.02
base_confidence: 0.92
lifecycle: stable
lifecycle_changed: 2026-10-10
created: 2026-10-03
updated: 2026-10-10
---

# §06 Java 8-25 版本时间线 + 速查表

## 1. 版本总览（Java 8 → Java 25）

| 版本 | 发布日期 | LTS | EOL | 主要主题 |
|---|---|---|---|---|
| **Java 8** | 2014-03 | ✅ | 2026-12 (个人) / 2030-12 (商业) | Lambda / Stream / Optional / `java.time` |
| Java 9 | 2017-09 | ❌ | 已 EOL | 模块系统(JPMS) / JShell / List.of |
| Java 10 | 2018-03 | ❌ | 已 EOL | `var` / 不可变集合 |
| **Java 11** | 2018-09 | ✅ | 2026-09 (免费) / 2032-01 (商业) | HTTP Client / 单文件启动 / String 增强 |
| Java 12 | 2019-03 | ❌ | 已 EOL | switch 表达式 (preview) |
| Java 13 | 2019-09 | ❌ | 已 EOL | text blocks (preview) |
| Java 14 | 2020-03 | ❌ | 已 EOL | records (preview) / instanceof pattern (preview) |
| Java 15 | 2020-09 | ❌ | 已 EOL | sealed (preview) / text blocks (final) |
| Java 16 | 2021-03 | ❌ | 已 EOL | records (final) / instanceof pattern (final) |
| **Java 17** | 2021-09 | ✅ | 2026-09 (免费) / 2030-09 (商业) | sealed (final) / 强封装 |
| Java 18 | 2022-03 | ❌ | 已 EOL | 简单 web server / UTF-8 默认 |
| Java 19 | 2022-09 | ❌ | 已 EOL | virtual threads (preview 1) / structured concurrency (incubator) |
| Java 20 | 2023-03 | ❌ | 已 EOL | virtual threads (preview 2) |
| **Java 21** | 2023-09 | ✅ | 2026-09 (免费) / 2031-09 (商业) | virtual threads (final) / pattern switch (final) / record patterns (final) / sequenced collections |
| Java 22 | 2024-03 | ❌ | 2025-09 | unnamed variables / FFM API / statements before super (preview) / stream gatherers (preview) |
| Java 23 | 2024-09 | ❌ | 2025-09 | primitive patterns (preview 1) / module imports (preview 1) |
| Java 24 | 2025-03 | ❌ | 2025-09 | stream gatherers (final) / AOT class loading (Leyden) |
| **Java 25** | 2025-09-16 | ✅ | 2030-09 | module imports (final) / compact source files (final) / flexible constructor bodies (final) / scoped values (final) / structured concurrency (5th preview) |

## 2. LTS 节奏演进

```
Java 8  (2014) ── 6 年 ──> Java 11 (2018)
Java 11 (2018) ── 3 年 ──> Java 17 (2021)
Java 17 (2021) ── 2 年 ──> Java 21 (2023)
Java 21 (2023) ── 2 年 ──> Java 25 (2025) ← 当前最新 LTS
```

**节奏从 6 年 → 3 年 → 2 年**：Oracle 从 Java 11 起改为更短的 LTS 节奏，让企业升级更灵活。

**支持窗口**：

| LTS | 免费支持 (Oracle) | 商业支持 (Oracle) | 第三方扩展（如 Azul）|
|---|---|---|---|
| Java 8 | 已结束 | 2030-12 | 至 2030+ |
| Java 11 | 2026-09 | 2032-01 | 至 2032+ |
| Java 17 | 2026-09 | 2030-09 | 至 2030+ |
| Java 21 | 2026-09 | 2031-09 | 至 2031+ |
| **Java 25** | 2030-09 | 2035-09 | 2030+ |

## 3. 语言特性 timeline（按类别）

### 数据载体演进
```
Java 14: records (preview 1)
Java 15: records (preview 2)
Java 16: records (final) ✓
Java 17: sealed classes (final) ✓
Java 21: record patterns (final) ✓
```

### 模式匹配演进
```
Java 14: switch 表达式 (final) + instanceof pattern (preview 1)
Java 15: sealed classes (preview 1) + text blocks (final)
Java 16: instanceof pattern (final) ✓
Java 17: switch pattern (preview 1) + sealed classes (final) ✓
Java 21: switch pattern (final) ✓ + record patterns (final) ✓ + unnamed patterns (preview)
Java 22: unnamed variables (final) ✓
Java 25: primitive patterns (preview 3)
```

### 并发演进
```
Java 5:   Future + ExecutorService
Java 7:   ForkJoinPool
Java 8:   CompletableFuture + parallelStream
Java 9:   Reactive Streams Flow API
Java 19:  virtual threads (preview 1) + structured concurrency (incubator)
Java 21:  virtual threads (final) ✓ + scoped values (preview)
Java 22:  FFM API (final) ✓
Java 24:  virtual threads pinning 改进（JEP 491）
Java 25:  scoped values (final) ✓ + structured concurrency (5th preview) + stable values (preview)
```

### 模块化演进
```
Java 9:   JPMS (module-info.java) 强封装
Java 25:  Module Import Declarations (final) ✓  ← 教学友好
```

### 教学体验演进
```
Java 8:   Lambda / Stream / 接口 default
Java 11:  单文件启动 (java Hello.java)
Java 14:  records (样板代码 -60 行)
Java 21:  unnamed patterns (变量声明简化)
Java 25:  Compact Source Files + Instance Main Methods (Hello World 从 8 行变 4 行)
```

### 性能/运行时演进
```
Java 9:   G1 默认 GC + 模块化启动优化
Java 10:  容器感知 (cgroup memory limits)
Java 11:  ZGC (experimental) + Epsilon (no-op GC)
Java 14:  ZGC 跨平台
Java 15:  ZGC (production) + Shenandoah (production)
Java 17:  Generational ZGC (大幅降低暂停)
Java 21:  Generational ZGA 默认
Java 22:  G1 region pinning 修复
Java 24:  AOT class loading & linking (JEP 483)
Java 25:  Compact Object Headers (-22% 堆) + Generational Shenandoah + AOT ergonomics
```

## 4. 关键 JEP 速查表

| JEP | 标题 | 版本 | 状态 |
|---|---|---|---|
| 261 | Module System | Java 9 | final |
| 286 | Local Variable Type Inference (`var`) | Java 10 | final |
| 321 | HTTP Client | Java 11 | final |
| 330 | Single-File Source-Code Launch | Java 11 | final |
| 361 | Switch Expressions | Java 14 | final |
| 378 | Text Blocks | Java 15 | final |
| 394 | Pattern Matching for `instanceof` | Java 16 | final |
| 395 | Records | Java 16 | final |
| 409 | Sealed Classes | Java 17 | final |
| 420 | Pattern Matching for `switch` | Java 21 | final |
| 431 | Sequenced Collections | Java 21 | final |
| 440 | Record Patterns | Java 21 | final |
| 444 | Virtual Threads | Java 21 | final |
| 447 | Statements before `super(...)` | Java 22 | final (JEP 513) |
| 454 | Foreign Function & Memory API | Java 22 | final |
| 456 | Unnamed Variables & Patterns | Java 22 | final |
| 461 | Stream Gatherers | Java 24 | final |
| 481 | Scoped Values (Preview) | Java 21+ | Java 25 final |
| 485 | Stream Gatherers (final) | Java 24 | final |
| 502 | Stable Values (Preview) | Java 25 | preview 1 |
| 505 | Structured Concurrency | Java 25 | preview 5 |
| 506 | Scoped Values | Java 25 | final |
| 507 | Primitive Patterns | Java 25 | preview 3 |
| 511 | Module Import Declarations | Java 25 | final |
| 512 | Compact Source Files & Instance Main Methods | Java 25 | final |
| 513 | Flexible Constructor Bodies | Java 25 | final |

## 5. 升级路径建议

### 5.1 从 Java 8 升级到 Java 25

**路径 1（推荐）**：Java 8 → 17 → 25
- Java 17 是 LTS，先踩一遍：records / sealed / switch 表达式 / text blocks
- 半年后再上 Java 25：virtual threads + sequenced collections + scoped values

**路径 2（激进）**：Java 8 → 21 → 25
- 直接用 virtual threads
- 但需要更多迁移工作（reflection / `--add-opens`）

**路径 3（保守）**：Java 8 → 11 → 17 → 25
- 保留 11 多一年（HTTP Client）
- 风险最小

### 5.2 各升级阶段的"必做清单"

**Java 8 → 11**：
- [ ] 替换 `javax.xml.bind`（已移除）→ `jakarta.xml.bind`
- [ ] 替换 `javax.annotation` → `jakarta.annotation`
- [ ] 替换 `sun.misc.*` 反射 → 公开 API 或 `--add-opens`
- [ ] JDK 11 默认 UTF-8（Java 8 默认 OS 编码）
- [ ] TLS 1.0/1.1 默认禁用

**Java 11 → 17**：
- [ ] 强封装（`--illegal-access` 已默认 deny）
- [ ] 替换 Nashorn JavaScript 引擎（已移除）→ GraalVM
- [ ] 替换 Corba / Java EE 模块
- [ ] 启用 records（POJO 替换）
- [ ] 启用 sealed class

**Java 17 → 21**：
- [ ] virtual threads（移除 synchronized 块，改为 ReentrantLock）
- [ ] pattern switch 替代 instanceof 链
- [ ] record patterns 替代 visitor 模式
- [ ] sequenced collections API 适配

**Java 21 → 25**：
- [ ] compact object headers（无需迁移，JVM 自动）
- [ ] AOT（可选优化）
- [ ] scoped values 替代 ThreadLocal
- [ ] module imports（教学/脚本用）

### 5.3 工具链要求

| Java 版本 | 最低工具链 |
|---|---|
| Java 8 | Maven 3.x / Gradle 4.x |
| Java 11 | Maven 3.5+ / Gradle 5+ |
| Java 17 | Maven 3.6+ / Gradle 7+ / Spring Boot 3.x |
| Java 21 | Maven 3.9+ / Gradle 8.5+ / Spring Boot 3.2+ |
| Java 25 | Maven 3.9.9+ / Gradle 8.10+ / Spring Boot 3.4+ / IntelliJ 2024.3+ |

## 6. 实战选型矩阵

| 场景 | 推荐 Java 版本 |
|---|---|
| **新项目（2026 起步）** | Java 25 (LTS) |
| 现有 Spring Boot 项目 | Java 21 (LTS) — virtual threads 兼容最好 |
| 保守型企业（金融/政府）| Java 17 (LTS) — 长期稳定 |
| 维护老系统 | 至少 Java 11 (有 HTTP Client) |
| Android | Java 17 起步（API 等级 33+）|
| 教学/学习 | Java 25（compact source files + module imports）|
| 高并发服务（I/O-bound）| Java 21+（virtual threads）|
| 极致低延迟（低 GC 暂停）| Java 25（Generational ZGC + Compact Headers）|

## 7. 反模式

❌ **跳过 LTS 直接用非 LTS 生产** — Java 22/23/24 没社区支持
❌ **用 Java 25 预览特性不评估** — 每次小版本可能破坏
❌ **virtual threads + synchronized** — pinning 问题（Java 24 才彻底解决）
❌ **records 当 JPA 实体** — JPA 需要可变 setter
❌ **sealed class + 大量 default 兜底** — 失去 exhaustive 优势
❌ **pattern switch 漏 default** — 即使你认为不会发生
❌ **Stream Gatherer 写得太花** — 自定义中间操作难调试
❌ **scoped value 替代 ThreadLocal 时机不对** — 老项目 ThreadLocal 是 OK 的
❌ **FFM 调 C 库但不管理 Arena** — 内存泄漏

## 8. Java 生态交叉引用

- **TypeScript** 对比: [[../typescript-analysis/summary]] — 同样有 union/intersection/pattern matching，TS 是结构化类型
- **Kotlin** 对比: [[../kotlin-analysis/summary]] — Kotlin 一直领先（sealed/data class 早于 Java 多年），Java 25 在 pattern matching 追平
- **gRPC 实战**: [[../grpc-analysis/summary]] — grpc-java 已用 virtual threads 重写
- **OkHttp 实战**: [[../okhttp-analysis/summary]] — OkHttp 5+ 适配 Java 21+ virtual threads
- **mu-server 实战**: [[../mu-server-2.4.2-analysis/summary]] — 内部使用虚拟线程 + Scoped Values

## 相关笔记

- **Java 8 基础**: [[draft-01-java-8-foundations]]
- **Java 9-11 现代化**: [[draft-02-java-9-11-modernization]]
- **Java 14-17 数据+模式**: [[draft-03-java-14-17-data-and-patterns]]
- **Java 21 LTS**: [[draft-04-java-21-lts-virtual-threads]]
- **Java 22-25 现代并发**: [[draft-05-java-22-25-modern-concurrency]]
- **综合入口**: [[summary]]
- **Map of Content**: [[moc]]
