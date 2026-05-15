---
title: "DB2 12工具增强"
category: concepts
tags: [db2, utilities, reorg, load, runstats, backup, recovery, zos]
sources: [下载/db2.pdf]
summary: DB2 12工具增强：REORG支持PBG分区创建和LOB防COPY-pending、LOAD RESUME BACKOUT、RUNSTATS USE PROFILE、备份恢复增强等。
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

# DB2 12工具增强

## 概述

DB2 12对关键数据库工具进行了大量增强，包括REORG、LOAD、RUNSTATS以及备份恢复功能，提升了运维效率和系统可用性。

## REORG增强

### PBG（Partition-by-Growth）增强

DB2 12的`REORG`工具改进以支持在分区级REORG期间**为溢出行创建新PBG分区**，改善数据可用性。

### FlashCopy管理改进

- **避免COPY-pending状态**：当REORG创建inline FlashCopy但FlashCopy失败时，DB2 12避免将page set置于COPY-pending状态
- REORG返回代码8表示FlashCopy未成功完成

### LOB表空间防COPY-pending

新功能防止在PBG REORG期间LOB表空间进入COPY-pending限制状态：
- 为新建的LOB表空间分配inline image copy
- 确保LOB数据一致性

### 空PBG分区删除

REORG支持删除空PBG分区：

```sql
REORG TABLESPACE tsname EMPTY_PGSPARTS
```

### COMPRESSRATIO目录列支持

REORG支持新的`COMPRESSRATIO`目录列。

### 分区级REORG改进

改进分区级PBG REORGs，允许在REORG期间处理更多场景。

### REORG排水（drain）失败显示

REORG显示每个drain失败时的声称者信息，便于诊断。

## RUNSTATS增强

### 无需COUNT的FREQVAL指定

```sql
RUNSTATS TABLESPACE dbname.tsname
  INDEX(indexname)FREQVAL NUMCOLS 3
```

替代之前需要`COUNT n`指定的方式。

### USE PROFILE支持

统计信息profile现在不仅可用于`RUNSTATS`工具，还可用于`LOAD`和`REORG TABLESPACE online`工具收集inline统计信息。

### INVALIDATECACHE选项

对于收集inline统计信息的工具，可选择**不使动态语句缓存中的语句失效**：

```sql
RUNSTATS TABLESPACE dbname.tsname
  USE PROFILE INVALIDATECACHE YES
```

**优势**：DB2可重用先前确定的访问路径，避免重新优化开销。

### TABLESPACE LIST INDEX改进

`RUNSTATS TABLESPACE LIST INDEX`功能改进。

### REGISTER关键字

RUNSTATS utility引入新的`REGISTER`关键字。

## LOAD和UNLOAD增强

### LOAD RESUME YES BACKOUT YES

LOAD utility支持`RESUME YES BACKOUT YES`功能：

- 如果任何输入记录有违规，加载的所有行都被删除
- LOAD完成时表空间可用
- `BACKOUT`或`BACKOUT YES`仅允许`SHRLEVEL NONE`
- 不允许与`INCURSOR`一起使用

### DRDA Fast Load

DB2 12增强了客户端驱动程序以支持**远程加载**到DB2 for z/OS服务器：
- CLI API和CLP被修改为向加载过程连续流式传输数据块
- 数据提取并传递给LOAD utility的任务100%可由zIIP卸载
- 显著减少分布式客户端加载数据的elapsed时间

## 备份和恢复增强

### 顺序image copy增强

增强`COPY`工具以支持更好的备份策略。

### FASTREPLICATION支持

COPY工具支持`FASTREPLICATION`选项。

### 备用copy池

支持为系统级备份定义**alternate copy pools**。

### FLASHCOPY_PPRCP关键字

增强FlashCopy相关操作。

### 时间点恢复增强

#### PARALLEL(1)默认

`RECOVER`工具默认使用`PARALLEL(1)`选项处理单个对象，提高恢复性能。

#### SCOPE UPDATED关键字

`RECOVER`工具使用`TORBA`或`TOLOGPOINT`选项时可应用`SCOPE UPDATED`关键字：

- 跳过自恢复点以来未更改的对象
- 避免恢复不必要的数据集
- 加快整体恢复速度

### MODIFY RECOVERY增强

增强`MODIFY RECOVERY`工具以更好地管理恢复历史。

## 相关概念

- [[references/ibm-db2-12-zos-technical-overview]] — 原始红皮书来源
- [[concepts/db2-12-scalability-availability]] — 可扩展性与高可用特性
- [[concepts/db2-12-performance-enhancements]] — 性能增强特性

## 来源

- IBM Redbooks SG24-8383-00, Chapter 11 "Utilities", December 2016