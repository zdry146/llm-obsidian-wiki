---
title: "TypeScript 全量分析 - MOC (Map of Content)"
category: synthesis
tags: [typescript, javascript, type-system, generics, node-js, frontend, analysis, moc, index]
sources:
  - "TypeScript 5.x Docs (typescriptlang.org/docs)"
  - "TypeScript Handbook"
  - "Microsoft TypeScript Blog"
  - "Effective TypeScript (Dan Vanderkam)"
summary: "TypeScript 5.x 全量分析 - 类型系统 / 函数 / 异步 / 泛型 / 类 / 模块 / 工具链 / 构建 / 与 JS/Flow/Dart 对比 / 在 Node.js 与前端生态中的角色"
provenance:
  extracted: 0.90
  inferred: 0.08
  ambiguous: 0.02
base_confidence: 0.88
lifecycle: draft
lifecycle_changed: 2026-09-20
created: 2026-09-20
updated: 2026-09-20
---

# TypeScript 全量分析 - Map of Content

> 本目录是 TypeScript **5.x** 的全量分析，涵盖类型系统、泛型、条件类型、异步、类、模块、工具链、构建、与 JavaScript/Flow/Dart 对比，最后说明 TypeScript 在 Node.js / React / 全栈生态中的角色。

## 主入口
- **[[summary|综合报告]]** — 执行摘要 + 生态位 + 13 章导航 + 与 JS/Flow/Dart 对比

## 13 章深度分析

| § | 子页面 | 核心内容 |
|---|---|---|
| **§00** | **[[draft-00-background\|背景与生态位]]** | Microsoft 出品、JavaScript 超集、C# 风格、Node.js/前端生态 |
| §01 | **[[draft-01-syntax-core\|核心语法与类型基础]]** | 类型注解、基础类型、字面量类型、类型推断、any vs unknown |
| §02 | [[draft-02-functions\|函数与异步]] | 函数类型、可选参数、默认参数、async/await、Promise |
| §03 | **[[draft-03-type-system\|类型系统（核心）]]** | 联合类型、交叉类型、类型缩窄、字面量类型、type guard |
| §04 | **[[draft-04-generics\|泛型与高级类型]]** | 泛型函数/类、约束、条件类型、映射类型、infer 关键字 |
| §05 | [[draft-05-classes-oop\|类与面向对象]] | class、修饰符、abstract、implements、装饰器 |
| §06 | [[draft-06-modules\|模块系统]] | ESM / CommonJS、import/export、type-only、命名空间 |
| §07 | [[draft-07-tooling\|工具链]] | tsc、ts-node、tsx、ESLint、Prettier、IDE |
| §08 | [[draft-08-build-bundlers\|构建与打包]] | tsc / webpack / vite / esbuild / swc 对比 |
| §09 | [[draft-09-best-practices\|最佳实践]] | 13 条铁律 + tsconfig 配置 + 类型设计原则 |
| §10 | [[draft-10-known-issues\|已知坑]] | any 滥用、类型断言、装饰器实验性、严格模式 |
| §11 | **[[draft-11-comparison\|对比选型]]** | **TypeScript vs JavaScript vs Flow vs Dart** |
| §12 | [[draft-12-use-cases\|应用场景]] | Node.js 后端 / React/Vue/Angular 前端 / 全栈 / 库开发 |
| **§13** | **[[draft-13-typescript-vs-javascript\|TS vs JS 深度对比]]** | **13 维 side-by-side + 何时用 + JS→TS 迁移指南** |

## 标签
`#typescript` `#javascript` `#type-system` `#generics` `#node-js` `#frontend`

## 元信息

- **分析对象**: TypeScript 5.x (2024+)
- **作者**: Microsoft（Anders Hejlsberg 领导）
- **协议**: Apache 2.0
- **目标**: 编译到 JavaScript（ES3+）
- **运行时**: Node.js / 浏览器 / Deno / Bun
- **分析时间**: 2026-09-20
- **执行者**: Spark (基于 TypeScript 官方文档 + 实战)
- **总产出**: 1 moc + 1 summary + 13 draft ≈ 4000 行 markdown

## 跨笔记链接

- **JVM 网络生态**：本项目以 JVM 为主，TypeScript 是 JavaScript 生态，**直接 cross-link 少**
- **对照参考**：[[grpc-analysis/summary]] 中的 grpc-web（TypeScript 实现）是 TS 跨生态例子
- **互补**：TypeScript 后端（Node.js / Deno）与 Kotlin 后端（[[kotlin-analysis/draft-12-use-cases|JVM Server]]）形成全栈对照