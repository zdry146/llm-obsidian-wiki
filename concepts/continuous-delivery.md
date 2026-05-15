---
title: "DB2 12连续交付与函数级别"
category: concepts
tags: [db2, continuous-delivery, function-level, zos]
sources: [下载/db2.pdf]
summary: DB2 12引入连续交付模式，用函数级别（Function Level）替代迁移模式，通过ACTIVATE命令控制新功能启用时机。
provenance:
  extracted: 0.80
  inferred: 0.15
  ambiguous: 0.05
base_confidence: 0.65
lifecycle: draft
lifecycle_changed: 2026-05-14
created: 2026-05-14
updated: 2026-05-14
---

# DB2 12连续交付与函数级别

## 概述

DB2 12是**首个采用连续交付（Continuous Delivery）模式的DB2版本**。在此之前，DB2通过版本迁移（migration modes）来管理新功能发布：CM（转换模式）→ CM*（星模式）→ ENFM（启用新功能模式）→ NFM（新功能模式）。这种方式周期长、客户无法控制新功能启用时间。

连续交付通过**函数级别（Function Level）**机制，允许客户：
- 在服务流中持续获取新功能和修复，无需等待完整新版本
- 通过`ACTIVATE`命令**自行控制**何时启用新功能
- 在稳定系统需求和功能灵活性之间取得平衡

## 函数级别详解

### 格式

函数级别采用9字节字符串格式：`VvvRrrMmmm`
- `vv` = version（版本）
- `rr` = release（发行版）
- `mmm` = modification level（修改级别）

### DB2 12初始函数级别

| 函数级别 | 含义 |
|---------|------|
| **V12R1M100** | DB2 12 GA代码级别，与DB2 11功能兼容。可类比为"转换模式（CM）"。DB2 11成员可共存，可回退到DB2 11 |
| **V12R1M500** | DB2 12最低新功能级别（等同于DB2 11的NFM）。DB2 11成员不可启动，不可回退到DB2 11 |
| **V12R1M100*** | 星函数级别（star function level）—— 已激活新功能后回退到的级别 |

### 新函数级别激活流程

1. 应用维护（PTF）后，代码级别更新（如 V12R1M501）
2. 部分级别需要运行 `CATMAINT` 升级目录级别
3. 执行 `ACTIVATE V12R1M501` 激活新函数级别
4. 应用包需要通过 `APPLCOMPAT` 选项与函数级别对应

## 目录级别（Catalog Level）

目录级别标识CATMAINT作业已对DB2目录进行了特定级别转换。新格式为9字节：`VvvRrMmmm`

- 初始迁移到DB2 12必须执行 `CATMAINT UPDATE LEVEL V12R1M500`
- 这称为**单阶段目录迁移**（single-phase catalog migration），减少了迁移窗口
- 如果目录未达到最低要求级别，`ACTIVATE`命令会失败

## 代码级别（Code Level）

代码级别针对每个成员（member），因为维护可以不同方式应用到每个成员。格式为6字节字符串，如 `121500`（代表V12R1M500）。

## DISPLAY GROUP命令输出变化

DB2 11显示3字节目录级别，DB2 12改为9字节格式。输出现在始终显示三个函数级别：
- **当前函数级别**（Current Function Level）
- **最高已激活函数级别**（Highest Activated Function Level）
- **最高可能函数级别**（Highest Possible Function Level）

## 星函数级别（Star Function Level）

当激活低于当前级别的函数级别时，激活的是星函数级别。星函数级别下：
- 之前在高函数级别创建的对象、包、结构仍然可访问
- 这些对象**不能被删除和重新创建**
- 包仍然可以执行、重新绑定、自动绑定
- 新绑定的包只能在当前函数级别及以下的APPLCOMPAT级别绑定

## 应用兼容性（APPLCOMPAT）

应用必须通过 `APPLCOMPAT` bind选项指定对应的应用兼容性值，才能使用新函数级别的功能。这确保了现有应用在切换函数级别时的行为一致性。

## 关键概念

- [[references/ibm-db2-12-zos-technical-overview]] — 原始红皮书来源
- [[concepts/db2-12-performance-enhancements]] — 性能增强特性
- [[concepts/db2-12-sql-enhancements]] — SQL增强特性

## 来源

- IBM Redbooks SG24-8383-00, Chapter 2 "Continuous delivery", December 2016