---
title: "DB2 12安装与迁移"
category: concepts
tags: [db2, installation, migration, zos, z-osmf, prereqs]
sources: [下载/db2.pdf]
summary: DB2 12安装和迁移增强：单阶段目录迁移、无需SYSADM、z/OSMF支持、先决条件（处理器、软件、编程语言要求）。
provenance:
  extracted: 0.72
  inferred: 0.23
  ambiguous: 0.05
base_confidence: 0.65
lifecycle: draft
lifecycle_changed: 2026-05-14
created: 2026-05-14
updated: 2026-05-14
---

# DB2 12安装与迁移

## 概述

DB2 12显著简化了安装和迁移流程，支持单阶段目录迁移、无需SYSADM权限的安装，以及通过z/OS Management Facility进行基于Web的安装管理。

## 先决条件

### 数据共享要求

迁移到DB2 12的数据共享组需要所有成员同时迁移或同时安装DB2 12。

### 处理器要求

DB2 12支持多种IBM z Systems处理器架构。

### 软件要求

需要特定级别的z/OS操作系统和相關軟件。

### DB2 Connect先决条件

DB2 Connect产品需要满足特定版本要求才能连接DB2 12。

### 编程语言最低版本要求

| 语言 | 要求 |
|------|------|
| COBOL | 特定版本级别 |
| PL/I | 特定版本级别 |
| C/C++ | 特定版本级别 |
| Java | JDBC 4.2+ |

### 最小配置（IEASYSxx）

安装需要指定最小系统配置参数。

## 单阶段迁移和函数级别

### DB2 11的两阶段迁移问题

DB2 11需要两次目录转换：
1. 迁移到转换模式时转换目录
2. 启用新功能模式时再次转换

每次目录更新都会导致应用程序访问目录时出现资源争用。

### DB2 12的单阶段迁移

DB2 12通过单阶段目录迁移改善可用性：

- 初始迁移到DB2 12必须执行`CATMAINT UPDATE LEVEL V12R1M500`
- 这是唯一将目录转换为新功能的步骤
- **减少变更窗口**并提高应用程序工作负载的可用性
- V12R1M500目录级别与DB2 11兼容，支持共存和回退情况

## Fallback SPE

DB2 12支持回退到早期版本的特殊操作（Special Permission Equivalent, SPE），但有一定限制。

## EARLY代码

DB2 12引入了"EARLY代码"概念，支持在激活新函数级别之前使用部分DB2 12功能。

## 预迁移检查

### 检查项目

- 验证当前DB2版本和环境
- 检查应用程序兼容性
- 确认所有先决条件满足
- 验证DSNZPARM设置

### 创建和验证DSNZPARM及DECP模块

迁移过程包括创建新的DSNZPARM和DECP模块。

### 创建和验证DB2提供的例程

运行DSNTIJRT作业创建例程。

## REBIND at each new release

每次迁移到新版本后，建议对所有包进行REBIND以获得最佳性能。

## 激活新函数级别

### 激活流程

1. 确保所有数据共享组成员具有代码级别V12R1M500
2. 目录已迁移到V12R1M500级别
3. 执行`ACTIVATE V12R1M500`
4. 之后DB2 11成员无法启动
5. 无法回退到DB2 11

### 激活前测试

```sql
-ACTIVATE V12R1M500 TEST
```

可测试子系统是否满足激活要求。

## 无需SYSADM的安装或迁移

DB2 12允许在没有SYSADM权限的情况下安装或迁移，降低管理复杂度。

## z/OS Management Facility（z/OSMF）支持

### 概述

DB2 12支持使用z/OSMF进行基于Web的安装和迁移管理，提供更灵活的管理界面。

### 工作流程

1. 使用DB2安装CLIST和面板生成z/OSMF artifacts
2. 将生成的artifacts提供给z/OSMF
3. 通过Web界面完成安装或迁移任务

### 优势

- 从任何计算机管理安装
- 减少手工操作错误
- 支持远程管理

## 时间目录（Temporal Catalog）

### 系统周期数据版本控制

DB2 12的两个RTS（Real-Time Statistics）目录表支持系统周期数据版本控制。

### 迁移期间的实时统计外部化

迁移过程包括实时统计的外部化处理。

## 废弃和移除

### 废弃功能

- `NEWFUN`处理选项（被`SQLLEVEL`替代）
- 部分旧版本参数

### 移除功能

DB2 12移除了早期版本中存在的某些功能。

## 相关概念

- [[references/ibm-db2-12-zos-technical-overview]] — 原始红皮书来源
- [[concepts/continuous-delivery]] — 连续交付与函数级别管理
- [[concepts/db2-12-administrator]] — 管理员功能

## 来源

- IBM Redbooks SG24-8383-00, Chapter 12 "Installation and migration", December 2016