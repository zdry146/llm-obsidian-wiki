---
title: "DB2 12 SQL增强"
category: concepts
tags: [db2, sql, triggers, arrays, temporal-tables, merge, zos]
sources: [下载/db2.pdf]
summary: DB2 12在SQL方面的主要增强包括：SQL PL触发器支持、数组类型扩展、分页支持、MERGE语句增强、时间维度表增强。
provenance:
  extracted: 0.78
  inferred: 0.17
  ambiguous: 0.05
base_confidence: 0.63
lifecycle: draft
lifecycle_changed: 2026-05-14
created: 2026-05-14
updated: 2026-05-14
---

# DB2 12 SQL增强

## 概述

DB2 12在SQL语言能力上有多项重要扩展，涵盖触发器、数组类型、分页、MERGE语句和时间维度表等核心领域。

## 触发器增强（Additional Support for Triggers）

### SQL PL触发器

DB2 12允许使用**SQL编程语言（SQL PL）**编写触发器。SQL PL允许开发者使用IF语句、WHILE语句、GOTO语句等逻辑结构来构建触发器。

关键能力：
- 支持错误条件处理（handlers）
- 支持调试、部署和维护多个版本的触发器
- 几乎所有可在SQL PL routines中实现的功能都可用于增强触发器支持

### 基本触发器 vs 高级触发器

| 特性 | 基本触发器 | 高级触发器（SQL PL） |
|------|-----------|-------------------|
| 语言 | 传统SQL | SQL PL |
| 逻辑控制 | 有限 | 完整（IF/WHILE/GOTO等） |
| 错误处理 | 无 | 有（handlers） |
| 多版本支持 | 差 | 强 |
| 调试能力 | 差 | 强 |

### 触发器激活顺序维护

DB2 12支持维护触发器的激活顺序，确保触发器按预期顺序执行。

## 数组支持增强（Additional Support for Arrays）

### 数组作为全局变量

数组数据类型可以指定为**全局变量**，突破了之前只能在SQL PL routines中使用数组的限制。

- 现在可以在SQL PL routines之外引用数组
- 数组数据类型不再局限于SQL PL routines

### ARRAY_AGG增强

- **关联数组支持**：ARRAY_AGG聚合函数支持关联数组
- **可选ORDER BY子句**：ARRAY_AGG支持可选的ORDER BY子句

## 分页支持（Pagination Support）

DB2 12增强了分页能力：

1. **返回行子集**（Returning a subset of rows）
2. **数据相关分页支持**（Data-dependent pagination support）
3. **数字分页**（Numeric-based pagination）

这对于需要大量数据浏览的应用特别有用。

## MERGE语句增强

MERGE语句在DB2 12中得到多项增强：

| 增强项 | 说明 |
|-------|------|
| 额外源值支持 | 支持更灵活的数据源 |
| 额外数据修改支持 | 支持更多UPDATE/INSERT变体 |
| 额外匹配条件选项 | 更灵活的条件判断 |
| 额外谓词支持 | 在匹配条件上支持额外谓词 |
| 原子性 | 确保MERGE操作的原子性 |

## 时间维度表增强（Temporal Table Enhancements）

### 应用期间增强

- **参照约束支持**：可对包含BUSINESS_TIME期间的应用周期时间表强制参照约束
- **包含/包含模型**：支持start time和end time都包含在时间间隔内的模型（inclusive/inclusive model）

### 系统周期时间表逻辑事务

提供更大的数据修改灵活性：
- 支持更复杂的审计场景
- 增强了"谁在何时做了什么修改"的审计能力

### 审计能力

时间维度表的审计能力非常受欢迎，该功能已retrofit到DB2 11 for z/OS。

## 内置函数增强

### 新增哈希标量函数

支持多种哈希算法。

### GENERATE_UNIQUE_BINARY

新增标量函数，用于生成唯一二进制值。

### VARCHAR_BIT_FORMAT

增强了VARCHAR_BIT_FORMAT标量函数。

### TIMESTAMP增强

TIMESTAMP标量函数增强。

### XMLMODIFY增强

XMLMODIFY标量函数增强。

## 相关概念

- [[references/ibm-db2-12-zos-technical-overview]] — 原始红皮书来源
- [[concepts/continuous-delivery]] — 连续交付与函数级别管理
- [[concepts/db2-12-performance-enhancements]] — 性能增强特性

## 来源

- IBM Redbooks SG24-8383-00, Chapter 6 "SQL", Chapter 7 "Application Enablement", December 2016