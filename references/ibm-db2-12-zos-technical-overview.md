---
title: "IBM DB2 12 for z/OS Technical Overview"
category: references
tags: [db2, ibm, zos, database, redbooks]
sources: [下载/db2.pdf]
summary: IBM官方红皮书，介绍DB2 12 for z/OS的核心新功能，包括连续交付模式、函数级别管理、性能增强、SQL增强和高可用特性。
provenance:
  extracted: 0.85
  inferred: 0.12
  ambiguous: 0.03
base_confidence: 0.67
lifecycle: draft
lifecycle_changed: 2026-05-14
created: 2026-05-14
updated: 2026-05-14
---

# IBM DB2 12 for z/OS Technical Overview

IBM DB2 12 for z/OS 是IBM推出的大型机数据库版本，代号Version 12.1，于2016年12月发布。这是**首个采用连续交付（Continuous Delivery）模式的DB2版本**，意味着新功能和修复可以在服务流中持续交付，而不必等待完整的新版本发布。

## 文档信息

- **来源**: IBM Redbooks SG24-8383-00
- **出版时间**: 2016年12月
- **作者**: Meg Bernal、Tammie Dang、Acacio Ricardo Gomes Pessoa
- **适用于**: DB2 12 for z/OS（Version 12）

## 核心主题速览

### 连续交付模式
DB2 12 引入了**函数级别（Function Level）**概念替代原有的迁移模式（CM/CM*/ENFM/NFM）。新函数在应用后默认休眠，通过 `ACTIVATE` 命令控制何时启用。[[references/ibm-db2-12-zos-technical-overview#连续交付模式]]

### 性能突破
- **超过1100万次/秒的非聚簇数据插入**
- 引入快速插入算法（Fast Insert Algorithm）
- 内存索引优化（FTB - Fast Traversal Block）
- UNION ALL和Outer Join性能增强

### SQL能力扩展
- SQL PL触发器支持
- 数组类型作为全局变量
- 分页支持增强
- MERGE语句增强
- 时间维度表（Temporal Tables）增强

### 高可用与可扩展
- 单表支持**256万亿行**（通过PBR RPN结构）
- 数据共享组自动对等恢复（Peer Recovery）
- 异步锁双工（Asynchronous Lock Duplexing）
- 支持大于4GB的活动日志数据集

## 相关概念

- [[concepts/continuous-delivery]] — 连续交付与函数级别管理
- [[concepts/function-level]] — 函数级别详解
- [[concepts/db2-12-performance-enhancements]] — 性能增强特性
- [[concepts/db2-12-sql-enhancements]] — SQL增强特性

## 章节结构

| Part   | 主题                                           |
| ------ | -------------------------------------------- |
| Part 1 | Overview（概述）                                 |
| Part 2 | Subsystem（子系统 - 可扩展性、可用性、数据共享）               |
| Part 3 | Application Functions（应用函数 - SQL、应用启用、连接和管理） |
| Part 4 | Operations and Performance（运维和性能）            |
| Part 5 | Appendixes（附录）                               |

## 来源

- [[references/ibm-db2-12-zos-technical-overview]] — IBM Redbooks SG24-8383-00, December 2016