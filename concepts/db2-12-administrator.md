---
title: "DB2 12管理员功能"
category: concepts
tags: [db2, administrator, plan-stability, runstats, zparm, zos]
sources: [下载/db2.pdf]
summary: DB2 12管理员功能增强：动态计划稳定性（Stabilized Dynamic SQL）、资源限制设施、延迟列变更、插入分区等。
provenance:
  extracted: 0.78
  inferred: 0.17
  ambiguous: 0.05
base_confidence: 0.65
lifecycle: draft
lifecycle_changed: 2026-05-14
created: 2026-05-14
updated: 2026-05-14
---

# DB2 12管理员功能

## 概述

DB2 12为数据库管理员提供了强大的管理功能增强，包括动态SQL计划稳定性、资源限制、延迟变更和插入分区等关键特性。

## 动态计划稳定性（Dynamic Plan Stability）

### 概述

动态计划稳定性允许将动态SQL语句的运行时结构持久化到DB2目录中，并在后续执行时重新加载，类似于静态SQL的管理方式。这解决了动态SQL访问路径可能随环境变化而不稳定的问题。

### 稳定化方法

1. 将EDM池中动态语句的运行时结构捕获并写入目录表
2. 后续执行时从目录表重新加载运行时结构
3. 如果稳定结构无效，语句自动重新优化

### 目录表

| 表名 | 说明 |
|------|------|
| `SYSIBM.DYNSTAB` | 存储稳定化的语句文本 |
| `SYSIBM.DYNSPackage` | 存储稳定化的包信息 |
| `SYSIBM.DYNSPackage_Copy` | 存储稳定化包的副本 |
| `SYSIBM.DYNSTABLES` | 包含稳定化信息的表定义 |

### INVALIDATECACHE选项

运行`RUNSTATS`或`LOAD`/`REORG TABLESPACE online`使用统计信息profile时，可选择不使动态语句缓存中的语句失效：

- **不失效**（默认）：DB2可重用先前确定的访问路径
- **失效**：强制重新优化

### FREE STABILIZED DYNAMIC QUERY子命令

用于释放稳定化的动态查询：

```sql
FREE STABILIZED DYNAMIC QUERY stmtid
```

### EXPLAIN变化

EXPLAIN输出包含稳定化和哈希ID信息，用于追踪稳定化的语句。

### 监控

可通过IFCID 316监控稳定化活动。

## 资源限制设施（Resource Limit Facility, RLF）

### 反应式资源限制

DB2 12支持对**静态SQL**进行反应式资源限制控制：

- 当语句的估计成本超过资源限制时，在**绑定时**拒绝语句（而非运行时）
- 可通过`DSNTIPR`安装面板配置

### 使用场景

1. **防止高成本SQL进入生产**：在测试阶段识别有问题的查询
2. **细粒度控制**：按用户、计划或包设置限制
3. **保护系统稳定性**：防止资源耗尽导致系统不可用

## 延迟列变更（Column Level Deferred Alter）

### 概述

DB2 12支持对列的数据类型、长度、精度或尺度进行**延迟变更**，将表置于`AREOR`（advisory-REORG pending）状态而非限制性状态。

### 改进的可用性

在DB2 11中，某些列定义变更是立即变更，会导致受影响表上的索引进入限制状态。如果唯一索引进入限制状态，会导致表不可用。

DB2 12通过延迟变更避免此问题，应用程序可继续访问表和索引。

### ALTER TABLE列变更语法

```sql
ALTER TABLE myTable
  ALTER COLUMN col1 SET DATA TYPE VARCHAR(100);
```

变更通过后续在线`REORG TABLESPACE`或`REORG INDEX`生效。

## 插入分区（Insert Partition）

### 概述

DB2 12允许通过`ALTER`语句在表的中间动态添加分区，大大改善了对象可用性——对象无需被删除、重新创建和重新填充数据。

### ALTER ADD PARTITION

```sql
ALTER TABLESPACE myts
  ALTER ADD PARTITION;
```

### 受影响的工具

- LOAD
- REORG
- RECOVER
- CHECK DATA

### 目录变化

`SYSIBM.SYSTABLESPACE`等目录表增加新列以支持插入分区功能。

## 新增和废弃的子系统参数

### 新增参数（部分）

| 参数 | 说明 |
|------|------|
| `PEER_RECOVERY` | 控制数据共享对等恢复参与方式 |
| `PAGESET_PAGENUM` | 控制创建range-partitioned表空间时是否使用相对页号 |
| `DEFAULT_INSERT_ALGORITHM` | 设置默认插入算法（1=传统，2=快速插入） |
| `PROFILE_AUTOSTART` | 控制子系统启动时是否自动启动profile监控 |
| `AUTH_COMPATIBILITY` | 控制授权兼容模式 |

### 移除的参数

- 多个与旧版本兼容相关的参数

### 废弃的参数

- `NEWFUN`（被`SQLLEVEL`替代）

## DISPLAY GROUP命令变化

DB2 12的`-DISPLAY GROUP`命令显示三个函数级别（9字节格式）：
- **当前函数级别**（Current Function Level）
- **最高已激活函数级别**（Highest Activated Function Level）
- **最高可能函数级别**（Highest Possible Function Level）

代码级别从3字节改为6字节格式（如`121500`表示V12R1M500）。

## 相关概念

- [[references/ibm-db2-12-zos-technical-overview]] — 原始红皮书来源
- [[concepts/db2-12-performance-enhancements]] — 性能增强特性
- [[concepts/continuous-delivery]] — 连续交付与函数级别管理

## 来源

- IBM Redbooks SG24-8383-00, Chapter 9 "Administrator function", December 2016