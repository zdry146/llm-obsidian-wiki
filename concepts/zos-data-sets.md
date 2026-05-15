---
title: z/OS Data Sets
category: concepts
tags: [ibm, zos, dataset, naming, vsam, dfsms]
sources: [下载/zOS Basics.pdf]
summary: z/OS数据集是命名的磁盘存储单元，数据集名称遵循8+44命名规则（qualifier最多8字符），记录格式分为F/V/U三种，catalog和VTOC管理数据集位置信息，DFSMS自动化存储管理。
provenance:
  extracted: 0.85
  inferred: 0.10
  ambiguous: 0.05
base_confidence: 0.57
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15T01:21:00Z
updated: 2026-05-15T01:21:00Z
---

# z/OS Data Sets

## 数据集的定义

**Data Set（数据集）** 是z/OS中命名的磁盘存储单元，类似于其他系统中的"文件"概念 ^[extracted]。

数据集与普通文件的区别：
- 数据集有**catalog**中的元数据记录位置
- 数据集有**record format**概念（定长/变长/未定义）
- 数据集由**access method**读写

## 数据集命名规则（DSN）

z/OS数据集名称采用**层级命名** ^[extracted]：

```
格式：qualifier1.qualifier2.[...].member-name
```

规则：
- 每个qualifier最多8个字符
- 总长度最多44个字符（不含单引号）
- qualifier间用"."分隔
- 示例：`USER.OBJ.COBOL.LIBRARY`

特殊数据集：
- `SYS1.LINKLIB` — 系统Link Library
- `USER.MY.COBOL` — 用户Cobol源代码

## Record Formats（记录格式）

数据集记录格式分为三种 ^[extracted]：

| 格式 | 说明 | 特性 |
|------|------|------|
| **F（Fixed）** | 定长记录 | 所有记录长度相同，节省存储空间 |
| **V（Variable）** | 变长记录 | 记录长度可不同，4字节长度前缀 |
| **U（Undefined）** | 未定义 | 字节流，无长度信息 |

**Record Length（记录长度）**：
- F格式：LRECL = 记录长度
- V格式：LRECL = 最大记录长度
- U格式：LRECL被忽略

## Block Format（块格式）

- **FB** — 定长记录块（Fixed Block），相同长度记录连续存放
- **VB** — 变长记录块（Variable Block）
- **U** — 无块（Unblocked）

Block Size（BLKSIZE）决定I/O缓冲区大小。

## 数据集类型

### 顺序数据集（Sequential Data Set）
- 最简单的数据集类型
- 顺序读写，可磁带存储
- 示例：日志文件、备份数据

### 分区数据集（PDS - Partitioned Data Set）
- 类似"目录"，包含多个member
- 每个member是独立的数据集
- 常用于源代码库（LOADLIB等）
- 成员可独立添加、删除、替换
- 缺点：删除成员后空间不回收，外部碎片

### PDSE（PDS Extended）
- PDS的增强版
- 动态空间管理，删除成员后空间回收
- 支持member-level安全控制
- 更好的性能

### VSAM数据集
- Virtual Storage Access Method管理的文件
- 详见：[[zos-io-data-management]]

## Catalogs和VTOC

### Master Catalog
- 记录系统中所有数据集的位置
- z/OS启动时自动打开
- 可被多个系统共享（通过ICF Catalog）

### VTOC（Volume Table of Contents）
- 每个磁盘卷上的索引
- 记录该卷上所有数据集的名称、位置、大小

设备地址命名示例：`132` = Channel 1, Control Unit 3, Device 2 ^[extracted]。

## DFSMS角色

DFSMS（Data Facility Storage Management Subsystem）自动管理数据集的存储 ^[extracted]：

- 根据数据集属性（如访问频率）自动分配存储
- 数据迁移（从磁盘到磁带）自动化
- 备份策略自动化

## 相关页面

- [[zos-io-data-management]] — 访问方法和I/O
- [[zos-jcl-basics]] — JCL中数据集的使用
- [[zos-overview]] — z/OS虚拟存储概念