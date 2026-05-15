---
title: JCL Data Set Disposition
category: concepts
tags: [ibm, zos, jcl, dataset, disp, catalog, sms]
sources: [下载/jcl user guide.pdf]
summary: 数据集DISP参数详解数据集的创建、保持、传递和删除，以及编目机制和SMS管理数据集的属性。
provenance:
  extracted: 0.88
  inferred: 0.08
  ambiguous: 0.04
base_confidence: 0.72
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15T01:35:00Z
updated: 2026-05-15T01:35:00Z
---

# JCL Data Set Disposition

## 概述

DISP参数控制数据集在作业步开始和结束时的状态，以及正常/异常结束时的处置方式 ^[extracted]。

## DISP三参数结构

`DISP=(status,normal-disposition,abnormal-disposition)`

| 参数位置 | 参数名 | 用途 |
|---|---|---|
| 第1个 | status | 数据集在作业步开始时的状态 |
| 第2个 | normal-disposition | 作业步正常结束时对数据集的处置 |
| 第3个 | abnormal-disposition | 作业步异常结束时对数据集的处置 |

### Status（状态）

| 值 | 含义 |
|------|------|
| `NEW` | 数据集将被创建，必须由作业步创建 |
| `OLD` | 数据集已存在，本作业步独占使用 |
| `SHR` | 数据集已存在，可与其它作业共享使用 |
| `MOD` | 数据集已存在，输出将追加到最后 |

### Normal和Abnormal Disposition（处置）

| 值 | 含义 |
|------|------|
| `CATLG` | 创建并编目数据集；如果数据集已存在则失败 |
| `KEEP` | 保留数据集（不编目）；如果数据集已编目则不能与KEEP一起使用NEW |
| `DELETE` | 删除数据集 |
| `PASS` | 将数据集传递给后续步骤（不编目） |
| 默认（省略） | 对于NEW数据集等同于DELETE |

### DISP参数示例

```jcl
//DD1 DD DSN=NEW.DATA,DISP=(NEW,CATLG,DELETE)
//DD2 DD DSN=OLD.DATA,DISP=OLD
//DD3 DD DSN=SHARED.DATA,DISP=(SHR,KEEP)
//DD4 DD DSN=TEMP.DATA,DISP=(NEW,PASS)
//DD5 DD DSN=OUTPUT.DATA,DISP=(NEW,CATLG,DELETE)
```

## 数据集完整性处理

当多个作业同时请求同一数据集时，系统根据以下规则授予控制权 ^[extracted]：

| 请求类型 | 数据集当前被使用（共享控制） | 数据集当前被使用（独占控制） | 数据集当前未使用 |
|---|---|---|---|
| 请求共享控制 | 授予 | 等待释放或降级时授予 | 授予 |
| 请求独占控制 | 等待释放 | 等待释放 | 授予 |

如果作业请求不可用的数据集，系统向操作员发出消息 `JOB jjj WAITING FOR DATA SETS`，作业等待直到所需数据集可用，除非操作员取消作业 ^[extracted]。

## 编目机制

### 编目（Catalog）

当DISP指定 `CATLG` 时：
- 数据集被创建并编目到系统catalog中
- 后续作业可通过数据集名称直接引用
- catalog条目包含数据集的卷位置、设备类型等信息

### 保持（KEEP）

当DISP指定 `KEEP` 时：
- 数据集被保留但不编目
- 后续引用必须指定卷信息
- 通常用于临时或不需长期保留的数据集

### 删除（DELETE）

当DISP指定 `DELETE` 时：
- 作业步结束时数据集被删除
- 不再保留任何引用

### 传递（PASS）

当DISP指定 `PASS` 时：
- 数据集可被同一作业中的后续步骤使用
- 后续步骤必须指定正确的DISP（如OLD或SHR）
- 作业结束时如果未被使用则删除

## SMS管理的数据集

当SMS（Storage Management Subsystem）活动时，数据集属性由以下构造定义 ^[extracted]：

| SMS构造 | 用途 |
|---|---|
| **Data Class** (DATACLAS) | 默认记录长度、块大小、记录格式等 |
| **Storage Class** (STORCLAS) | 性能特性、设备类型、备份策略等 |
| **Management Class** (MGMTCLAS) | 保留、备份、迁移策略等 |

### SMS数据集示例

```jcl
//SMSDS DD DSN=MY.SMS.DATA,DATACLAS=DCLAS1,STORCLAS=SCLAS1,DISP=(NEW,KEEP)
```

### SMS注意事项

- SMS管理的数据集在分配时编目（非SMS数据集在步骤结束时编目）
- SMS可以根据安装编写的ACS（自动类选择）例程自动选择存储类
- 临时数据集如果指定STORCLAS或ACS例程选择了存储类，则由SMS管理

## 后向引用（Backward References）

可以使用后向引用从早期DD语句复制数据集名称 ^[extracted]：

```jcl
//COPYDS DD DSNAME=*.DDNAME
//COPYDS DD DSNAME=*.STEPNAME.DDNAME
//COPYDS DD DSNAME=*.STEPNAME.PROCSTEPNAME.DDNAME
```

### 示例

```jcl
//STEP1 EXEC PGM=PROGA
//DD1 DD DSN=INPUT.DATA,DISP=OLD
//STEP2 EXEC PGM=PROGB
//DD2 DD DSNAME=*.STEP1.DD1  引用STEP1的DD1
```

## 数据集连接（Concatenation）

可以逻辑上连接（串联）顺序或分区数据集，方法是在除第一个外的所有DD语句中省略ddname ^[extracted]：

```jcl
//INPUT DD DSN=FGLIB,DISP=(OLD,PASS)
//DD DSN=GROUP2,DISP=SHR
```

数据按定义DD语句的相同顺序处理。

## 相关页面

- [[concepts/zos-data-sets]] — z/OS数据集和命名规则
- [[concepts/zos-io-data-management]] — VSAM和DFSMS
- [[concepts/jcl-job-proc-step]] — JOB/EXEC/DD语句详解
- [[references/zos-jcl-user-guide]] — IBM官方JCL用户指南

## 来源

- IBM z/OS 3.1 MVS JCL User's Guide, SA23-1386-60, Chapters 12-17 "Data set resources"