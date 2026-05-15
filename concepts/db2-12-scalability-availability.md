---
title: "DB2 12可扩展性与高可用"
category: concepts
tags: [db2, scalability, availability, partition, zos]
sources: [下载/db2.pdf]
summary: DB2 12在可扩展性方面实现单表256万亿行支持，通过PBR RPN结构消除分区数量与大小的耦合；高可用方面支持在线压缩变更、目录可用性改进。
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

# DB2 12可扩展性与高可用

## 可扩展性（Scalability）

### 单表256万亿行

DB2 12最重要的可扩展性增强是支持**单表最多256万亿行**。这是通过新的**partition-by-range (PBR) table space结构**实现的，采用相对页号（Relative Page Numbers, RPN）。

#### PBR RPN关键特性

- **消除分区数量与大小的耦合**：以往分区数与分区大小之间存在规则限制，PBR RPN结构打破这一限制
- **七字节记录标识符（RID）**：由2字节part number + 5字节page number组成
- **无需担心分区空间耗尽**：不再需要研究分区限制键规则，避免因分区满导致的应用中断

#### PBR RPN表空间特性

- Range-partitioned table space
- 使用相对页号而非绝对页号
- 支持更大的表空间规模

#### PBR RPN索引特性

- **分区索引**：按分区组织
- **非分区索引**：全局索引结构

### DB2内部闩锁争用缓解

减少内部latch争用，提升并发性能。

### Buffer Pool Simulation

支持buffer pool模拟，便于容量规划和性能测试。

### 支持大于4GB的活动日志数据集

突破传统4GB限制，支持更大的活动日志。

## 高可用性（Availability）

### 待定义变更的改进

#### 索引压缩属性变更

在DB2 12之前，变更索引压缩属性会使索引进入限制性的**REBUILD-pending (RBDP)**状态。DB2 12改为将索引置于**advisory-REORG pending (AREOR)**状态，允许应用在变更期间继续访问索引。

#### 列变更

支持列的延迟变更，同样置于AREOR状态。

### 目录可用性改进

- **动态SQL语句处理**：改善目录可用性
- **单阶段目录迁移**：减少迁移窗口，提升系统可用性

### PBG表空间时间点恢复限制解除

移除point-in-time recovery对partition-by-growth (PBG)表空间的限制。

### PBR RPN DSSIZE增加

支持更大的数据集合尺寸。

### 插入分区（Insert Partition）

支持动态添加分区，无需重建表。

### REORG增强

- **PBG REORG改进**
- **FlashCopy管理改进**
- **LOB表空间REORG期间防止COPY-pending**

### LOAD RESUME YES BACKOUT YES

支持LOAD操作的可恢复性。

### 时间点恢复加速

- 默认使用`PARALLEL(1)`选项
- `SCOPE UPDATED`关键字支持

### TRANSFER OWNERSHIP SQL语句

支持数据库对象所有权的转移。

### GRECP和LPL恢复自动重试

自动重试机制，提升恢复成功率。

## 数据共享增强（Data Sharing）

### 自动对等恢复（Peer Recovery）

当数据共享组的一个成员失败时，另一个成员可自动启动该失败成员的恢复过程，采用轻量模式重启。

### 异步锁双工（Asynchronous Lock Duplexing）

简化为IRLM锁结构启用异步双工，性能接近simplex模式，同时保持双结构的高可用性优势。

### 改进的锁规避检查

优化lock avoidance机制，减少不必要的数据一致性问题。

## 相关概念

- [[references/ibm-db2-12-zos-technical-overview]] — 原始红皮书来源
- [[concepts/continuous-delivery]] — 连续交付与函数级别管理
- [[concepts/db2-12-performance-enhancements]] — 性能增强特性

## 来源

- IBM Redbooks SG24-8383-00, Chapter 3 "Scalability", Chapter 4 "Availability", Chapter 5 "Data Sharing", December 2016