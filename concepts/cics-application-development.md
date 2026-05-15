---
title: CICS应用开发
category: concepts
tags: [cics, application-development, oltp, batch, cobol, programming-paradigm]
sources: [/home/openclaw/下载/CICS.pdf]
summary: CICS应用开发核心概念：批处理vs联机事务处理、三种编程范式（non-conversational、conversational、pseudo-conversational）、三层架构、CICS资源定义。
provenance:
  extracted: 0.80
  inferred: 0.15
  ambiguous: 0.05
base_confidence: 0.80
lifecycle: draft
lifecycle_changed: 2026-05-14
created: 2026-05-14
updated: 2026-05-14
---

# CICS应用开发

## 批处理 vs 联机事务处理

### 批处理

数据以顺序方式整批喂入程序，结果在所有记录处理完毕后才返回 ^[extracted]。这是1970年代之前的主流模式 ^[extracted]。

### 联机事务处理（OLTP）

用户直接交互，每次请求需要快速响应（通常亚秒级）。CICS是这类处理的开创性解决方案 ^[extracted]。

**关键区别：** 批处理中用户是文件/机器；OLTP中用户是活人 ^[extracted]。

## 三种编程范式

CICS支持三种事务处理风格 ^[extracted]：

### Non-conversational Transaction

开始和结束一起发生，一个迭代内完成。类似于API调用，结果立即可用 ^[extracted]。

**示例：** 查询账户余额 — 输入账号，立即返回余额 ^[extracted]。

### Conversational Transaction

需要多次输入，路径根据输入决定。每次响应后程序挂起，等待用户回复，所有资源被锁定 ^[extracted]。

**示例：** 菜单驱动的交互系统 ^[extracted]。

### Pseudo-conversational Transaction（最常用）

用户在响应前离开（比如接电话），程序在用户"思考"期间释放资源 ^[extracted]。

**实现方式：** 程序在每次响应后立即结束（`EXEC CICS RETURN`），状态数据保存在**通信区（COMMAREA）**中，下次用户响应时程序从断点恢复 ^[extracted]。

这是现代CICS应用最主流的编程风格 ^[inferred]。

## 三层应用架构

CICS应用按职责分为三层 ^[extracted]：

### 1. 展示服务层（Presentation Services Layer）

处理与用户的所有通信。好处是更换展示方式（如从终端到移动端）不影响底层业务逻辑 ^[extracted]。

现代趋势：展示层已越来越多地移出CICS，由更合适的平台处理（Web服务器、移动后端等） ^[inferred]。

### 2. 业务逻辑层（Business Logic Layer）

执行实际的数据处理工作。这是CICS的核心价值所在 — 虽然客户引入了Java、Node.js等新语言，但核心COBOL应用仍然保留在CICS中 ^[extracted]。

"将整个应用用另一种语言重写到另一个平台并达到同等性能——最终得到什么？一张大账单，和一个功能相同的东西。" ^[extracted]

### 3. 数据服务层（Data Services Layer）

数据存储和检索逻辑独立为单独模块，可按需替换存储后端 ^[extracted]。程序可以通过Web服务访问其他平台的数据，下层存储方式对业务逻辑程序透明 ^[inferred]。

## 工资单示例应用

本书使用一个工资单（Payroll）系统作为贯穿示例 ^[extracted]：

- **交易ID：** PAYR
- **展示程序：** PAYPGM — 收集用户输入，管理状态（pseudo-conversational）
- **业务逻辑程序：** PAYBUS — 实际的数据处理
- **数据结构：** VSAM KSDS文件

**典型ADD操作流程：**
1. 用户输入部门和员工号，按PF4发起ADD
2. PAYPGM调用PAYBUS检查记录是否已存在
3. PAYPGM解锁输入字段，用户输入新数据
4. 用户再次按PF4确认，PAYBUS执行真正的添加 ^[extracted]

这是一个两阶段的pseudo-conversational处理 ^[extracted]。

## CICS资源

CICS是**资源驱动的系统** — 大多数资源在使用前必须先定义到资源表中 ^[extracted]。

资源定义由CICS系统程序员负责，因为需要了解整个环境的设计和资源分布 ^[extracted]。

**常见资源类型：**
- **VSAM文件** — 可以放在同一Region或不同的Region
- **程序（Program）** — 可被CICS加载和调用
- **交易（Transaction）** — 触发程序的入口
- **终端（Terminal）** — 连接CICS的3270终端

**协作要求：** 应用程序员和系统程序员需要协作才能正确设置所有资源属性 ^[extracted]。但如果只是扩展现有应用且不需要新资源，应用开发人员可以独立完成 ^[extracted]。

## 内置调试工具

CICS提供30+内置交易（transactions）用于开发和测试 ^[extracted]：

| 交易 | 功能 |
|---|---|
| **CECI/CECS** | 命令解释器 — 可在终端直接执行EXEC CICS命令（后者仅语法检查） |
| **CEDF/CEDX** | 执行诊断设施 — 交互式调试，可在每个CICS命令处拦截 |
| **CEBR** | 浏览临时存储队列（TSQ）和瞬时数据队列（TDQ） |

CECI和CEDF是最常用的两个调试工具 ^[extracted]。

## 相关概念

- [[concepts/cics-overview]] — CICS概述与QoS
- [[concepts/cics-hello-world-cobol]] — Hello World代码对比
- [[concepts/cics-programming-cobol]] — COBOL编程详解（COMMAREA、展示/业务逻辑分离）
- [[references/modernizing-applications-ibm-cics]] — 原始红皮书来源

## 来源

- IBM Redbooks REDP-5628-00, Chapter 2 "IBM CICS application development", December 2020