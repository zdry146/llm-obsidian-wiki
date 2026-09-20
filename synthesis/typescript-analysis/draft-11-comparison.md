---
title: "TypeScript vs JavaScript vs Flow vs Dart 对比选型"
category: synthesis
tags: [typescript, javascript, flow, dart, comparison, decision-matrix]
sources:
  - "TypeScript 官方文档"
  - "Flow 官方文档 (flow.org)"
  - "Dart 官方文档"
  - "Octoverse 2024 (GitHub)"
summary: "TypeScript vs JavaScript vs Flow vs Dart 6 维对比 + 选型决策矩阵"
provenance:
  extracted: 0.88
    inferred: 0.10
    ambiguous: 0.02
base_confidence: 0.86
lifecycle: draft
lifecycle_changed: 2026-09-20
created: 2026-09-20
updated: 2026-09-20
---

# §11 TypeScript vs JavaScript vs Flow vs Dart 对比

## 1. 4 大语言定位

| 语言 | 作者 | 类型系统 | 运行时 | 定位 |
|------|------|----------|--------|------|
| **TypeScript** | Microsoft | 静态（结构化）| 编译到 JS | JavaScript 的超集 |
| **JavaScript** | Brendan Eich / TC39 | 动态 | V8 / 浏览器 | Web 原生语言 |
| **Flow** | Facebook | 静态 | 编译到 JS | JavaScript 的类型检查器 |
| **Dart** | Google | 静态（健全）| Dart VM / 编译到 JS | Flutter / 全平台 |

## 2. 6 维对比矩阵

### 2.1 类型系统

| 维度 | TypeScript | JavaScript | Flow | Dart |
|------|-----------|------------|------|------|
| **类型系统** | 静态 + 结构化 | 动态 | 静态 + 结构化 | 静态 + 名义 |
| **渐进类型** | ✅ | N/A | ✅ | ❌（强类型）|
| **类型推断** | ✅ | ❌ | ✅ | ✅ |
| **null 安全** | ✅ (strict) | ❌ | ✅ | ✅（强）|
| **联合类型** | ✅ | N/A | ✅ | ❌（用 sealed class）|
| **映射类型** | ✅ | ❌ | ✅ | ❌ |
| **健全性** | 不健全 | N/A | 不健全 | **健全**（sound）|

### 2.2 生态与采用

| 维度 | TypeScript | JavaScript | Flow | Dart |
|------|-----------|------------|------|------|
| **GitHub stars** | 100k+ | 200k+ | 22k+ | 17k+ |
| **npm 下载** | 2 亿+ 周 | 最大 | 萎缩 | N/A |
| **生产采用** | React / Vue / NestJS | 全部 | 较少（Meta 内部）| Flutter |
| **2026 趋势** | ↑↑ | → | ↓ | ↑（Flutter 带动）|

### 2.3 学习曲线

| 维度 | TypeScript | JavaScript | Flow | Dart |
|------|-----------|------------|------|------|
| **JS 基础** | 必须 | N/A | 必须 | 不必 |
| **上手时间** | 1-2 周 | 几小时 | 1-2 周 | 2-4 周 |
| **生态文档** | 极丰富 | 极丰富 | 较少 | 中等 |

### 2.4 运行时与跨平台

| 维度 | TypeScript | JavaScript | Flow | Dart |
|------|-----------|------------|------|------|
| **原生运行时** | ❌（编译到 JS）| ✅ | ❌（编译到 JS）| ✅ Dart VM |
| **浏览器** | ✅ | ✅ | ✅ | ✅（编译）|
| **Node.js** | ✅ | ✅ | ✅ | ❌ |
| **移动端** | ❌（需 RN/Expo）| ❌ | ❌ | ✅ Flutter |
| **桌面** | ⚠️（Electron/Tauri）| ⚠️ | ⚠️ | ✅ Flutter Desktop |
| **嵌入式** | ❌ | ❌ | ❌ | ❌ |

### 2.5 类型健全性（Soundness）

```typescript
// TypeScript：不健全（可绕过）
let value: number = "hello" as any as number;   // 编译过，运行时崩

// Dart：健全（编译过就保证运行时类型）
int value = "hello" as int;   // ❌ 编译错误
```

**核心洞察**：**TypeScript 不健全是为了 JS 互操作**，Dart 健全是为了类型安全。

### 2.6 性能

| 维度 | TypeScript | JavaScript | Flow | Dart |
|------|-----------|------------|------|------|
| **运行时** | 同 JS | 基线 | 同 JS | VM/Flutter 稍快 |
| **启动** | 同 JS | 快 | 同 JS | 较慢 |
| **内存** | 同 JS | 中 | 同 JS | 较高 |

## 3. TypeScript vs JavaScript 详解

```javascript
// JavaScript：动态类型
function add(a, b) {
    return a + b;
}
add(1, "2");      // "12"（字符串拼接，bug）
add(1, null);     // 1（null 转 0）
add(1, undefined);  // NaN
```

```typescript
// TypeScript：静态类型
function add(a: number, b: number): number {
    return a + b;
}
add(1, "2");      // ❌ 编译错误
add(1, 2);         // ✅ 3
```

**差异**：TS 是 JS 的超集，所有合法 JS 都是合法 TS。

## 4. TypeScript vs Flow 详解

```javascript
// Flow（Facebook 内部）
// @flow
function add(a: number, b: number): number {
    return a + b;
}
```

```typescript
// TypeScript
function add(a: number, b: number): number {
    return a + b;
}
```

**差异**：
- 语法几乎相同
- TypeScript：完整语言 + 编译器
- Flow：纯类型检查器（不影响运行时）

**2026 现状**：Flow 已基本废弃，TypeScript 是事实标准。

## 5. TypeScript vs Dart 详解

```typescript
// TypeScript：渐进类型
function fetchUser(id: string | null): Promise<User> {
    if (id === null) throw new Error("id is null");
    // 业务逻辑
}

// Dart：强类型 + null 安全
Future<User> fetchUser(String? id) async {
    if (id == null) throw Exception("id is null");
    // 业务逻辑
}
```

**核心差异**：
- **TypeScript**：渐进类型，可绕过（健全性弱）
- **Dart**：强类型 + null 安全（编译过就保证）

## 6. 决策矩阵

### 6.1 新项目

| 场景 | 推荐 |
|------|------|
| **Web 前端** | TypeScript（React/Vue/Angular）|
| **Node.js 后端** | TypeScript（NestJS / Express + tRPC）|
| **跨端（Web + 移动）** | Dart（Flutter）|
| **JS 库维护** | JavaScript + JSDoc（无需 TS）|
| **AI 脚本 / 一次性** | JavaScript（直接跑）|

### 6.2 老项目迁移

| 现有 | 推荐路径 |
|------|---------|
| JavaScript | 增量 TypeScript（先 1 个文件）|
| Flow | 迁移 TypeScript（Flow 工具支持）|
| Dart | 保留（除非 Web 需求）|

### 6.3 学习路径

| 当前水平 | 建议 |
|---------|------|
| **JS 开发者** | TypeScript（1 周上手）|
| **Java/C# 开发者** | TypeScript（1 周上手）|
| **移动端开发者** | Dart / Flutter |
| **脚本需求** | JavaScript |

## 7. 性能对比（基准）

| 操作 | TypeScript | JavaScript | Dart |
|------|-----------|------------|------|
| **启动** | 同 JS | 快 | 中（VM 启动慢）|
| **执行** | 同 JS | 基线 | +10-30%（VM 优化）|
| **内存** | 同 JS | 中 | 高（VM 开销）|
| **bundle** | 100-500KB | 50-200KB | Flutter 4MB+ |

⚠️ **基准因场景差异大**——具体数据仅供参考。

## 8. 一句话对比

| 语言 | 一句话 |
|------|--------|
| **TypeScript** | JavaScript 的现代化超集，生态最大，类型渐进 |
| **JavaScript** | Web 原生语言，动态灵活，生态最丰富 |
| **Flow** | Facebook 内部工具，已基本被 TypeScript 取代 |
| **Dart** | Flutter 首选，强类型 + null 安全 + 健全 |

## 9. 实战建议

### 9.1 Web 应用（首选 TypeScript）

```
TypeScript + Vite + React/Vue/Angular
```

### 9.2 跨端应用（首选 Dart）

```
Flutter + Dart
```

### 9.3 Node.js 后端

```
TypeScript + NestJS / Express + Prisma / TypeORM
```

### 9.4 AI/数据脚本

```
JavaScript / TypeScript（视团队）
```

## 10. 关键洞察

1. **TypeScript 是 Web 生态标准**——新项目默认
2. **Dart 是 Flutter 跨端标准**——非 Web 首选
3. **Flow 已基本死亡**——别选
4. **JavaScript 不会消失**——TS 编译目标
5. **类型健全性不同**——TS 不健全（渐进），Dart 健全

## 11. 与 JVM 生态对照

| 维度 | TypeScript | [[kotlin-analysis/summary\|Kotlin]] |
|------|-----------|--------|
| **生态** | JS / 前端 / Node.js | JVM / Android / Spring |
| **类型** | 结构化渐进 | 静态 + null 安全 |
| **健全性** | 不健全 | 健全 |
| **异步** | async/await | 协程 |
| **跨平台** | Web + Electron | KMP（iOS/Android/JVM）|

**实战组合**：TypeScript 前端 + Kotlin 后端（JVM）+ gRPC 通信。

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **类型系统**: [[draft-03-type-system]]
- **泛型**: [[draft-04-generics]]
- **应用场景**: [[draft-12-use-cases]]
- **Kotlin 对照**: [[kotlin-analysis/summary]]
- **综合入口**: [[summary]]