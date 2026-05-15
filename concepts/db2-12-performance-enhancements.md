---
title: "DB2 12性能增强"
category: concepts
tags: [db2, performance, zos, insert, query-optimization]
sources: [下载/db2.pdf]
summary: DB2 12在性能方面的重大突破：超过1100万次/秒插入、内存索引优化、快速插入算法、UNION ALL和Outer Join增强。
provenance:
  extracted: 0.75
  inferred: 0.20
  ambiguous: 0.05
base_confidence: 0.62
lifecycle: draft
lifecycle_changed: 2026-05-14
created: 2026-05-14
updated: 2026-05-14
---

# DB2 12性能增强

## 概述

DB2 12 for z/OS 在性能方面实现了重大突破，引入了多种先进内存数据库技术和查询优化技术。

## 核心性能突破

### 非聚簇数据插入性能

DB2 12 可实现**超过1100万次/秒的非聚簇数据插入**，这是通过引入**快速插入算法（Fast Insert Algorithm）**实现的：

- 提高吞吐量
- 潜在降低日志开销
- 减少某些操作的elapsed时间和CPU时间
- 仅对使用 `MEMBER CLUSTER` 的universal table space启用
- 可系统级启用或针对单个table space启用

### 内存索引优化（In-Memory Index Optimization）

DB2 12引入了**快速遍历块（Fast Traversal Block, FTB）**用于索引：

- FTB是内存优化的索引结构
- 允许DB2比传统page-oriented遍历更快地遍历FTB
- 适用于缓存在buffer pool中的索引

### UNION ALL和Outer Join增强

UNION ALL和Outer Join共享类似的性能挑战——两者都可能导致数据物化到workfile，造成：
- 显著性能下降
- 消耗workfile资源

DB2 12通过以下方式最小化物化：
- 从物化中修剪不必要的列
- 将谓词下推到lower query blocks
- 将ordering和fetch first counters下推到lower query blocks
- 重排outer join表以避免物化

### 谓词优化（Predicate Optimization）

持续优化谓词处理，提升整体查询性能。

### 执行时间自适应索引（Execution Time Adaptive Index）

能够在执行时自适应调整索引策略。

## DDL新属性

### CREATE/ALTER TABLESPACE的INSERTALG子句

DDL clause支持在CREATE TABLESPACE和ALTER TABLESPACE上指定INSERT算法：
- `INSERTALG=1` = 传统算法
- `INSERTALG=2` = 快速插入算法

相关ZPARM：`DEFAULT_INSERT_ALGORITHM`

### SYSIBM.SYSTABLESPACE新列

`INSERTALG` 列记录table space使用的插入算法。

## 相关概念

- [[references/ibm-db2-12-zos-technical-overview]] — 原始红皮书来源
- [[concepts/continuous-delivery]] — 连续交付与函数级别管理
- [[concepts/db2-12-sql-enhancements]] — SQL增强特性

## 来源

- IBM Redbooks SG24-8383-00, Chapter 13 "Performance", December 2016