---
title: JCL IBM-Supplied Utilities
category: concepts
tags: [ibm, zos, jcl, utilities, program, ieb, iec,ief]
sources: [下载/jcl user guide.pdf]
summary: IBM提供的实用程序概述：数据移动（IEBGENER、IEBCOPY）、排序（SORT）、数据更新（DFSORT）、以及其它常用实用程序。
provenance:
  extracted: 0.78
  inferred: 0.15
  ambiguous: 0.07
base_confidence: 0.70
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15T01:35:00Z
updated: 2026-05-15T01:35:00Z
---

# JCL IBM-Supplied Utilities

## 概述

z/OS提供了大量IBM-supplied实用程序，用于日常数据处理任务 ^[extracted]。本页面概述主要实用程序及其JCL使用方式。

## 数据移动实用程序

### IEBGENER — 数据集复制

```jcl
//COPYEXEC PGM=IEBGENER
//SYSPRINT DD SYSOUT=*
//SYSUT1 DD DSN=INPUT.DATA,DISP=SHR
//SYSUT2 DD DSN=OUTPUT.DATA,DISP=(NEW,CATLG,DELETE),
//    SPACE=(TRK,(10,5)),UNIT=SYSDA
//SYSIN DD *
  MODE CHANGE
  MAXNAME=1
  MAXLITS=1
/*
```

**功能**：
- 复制顺序数据集
- 生成VB格式输出（RECFM=VBN）
- 生成模块输入数据

### IEBCOPY — 数据集复制/合并

```jcl
//COPYEXEC PGM=IEBCOPY
//SYSPRINT DD SYSOUT=*
//IN1 DD DSN=PDS.SOURCE,DISP=SHR
//OUT1 DD DSN=PDS.TARGET,DISP=(NEW,CATLG,DELETE),
//    SPACE=(CYL,(10,5)),UNIT=SYSDA
//SYSIN DD *
  COPY INDD=IN1,OUTDD=OUT1
/*
```

**功能**：
- 复制PDS/PDSE（分区数据集）
- 压缩PDS
- 选择性复制成员

## 排序实用程序

### DFSORT/SORT

```jcl
//SORTEXEC PGM=SORT
//SYSOUT DD SYSOUT=*
//SYSIN DD *
  SORT FIELDS=(1,10,CH,A)
  RECORD TYPE=F,LENGTH=(80)
/*
//SORTIN DD DSN=INPUT.DATA,DISP=SHR
//SORTOUT DD DSN=OUTPUT.DATA,DISP=(NEW,CATLG,DELETE),
//    SPACE=(TRK,(10,5)),UNIT=SYSDA
```

**SORT控制语句**：
- `SORT FIELDS=(start,length,type,order)` — 排序键
- `INCLUDE COND=` — 条件包含记录
- `OMIT COND=` — 条件排除记录
- `OUTFIL` — 输出格式控制

### ICETOOL — DFSORT工具

```jcl
//TOOL EXEC PGM=ICETOOL
//TOOLMSG DD SYSOUT=*
//DFSMSG DD SYSOUT=*
//IN DD DSN=INPUT.DATA,DISP=SHR
//OUT DD DSN=OUTPUT.DATA,DISP=(NEW,CATLG,DELETE)
//TOOLIN DD *
  SORT FROM(IN) TO(OUT) USING(CTL1)
  DISPLAY FROM(IN) TITLE('Sample Data')
/*
```

## 数据集实用程序

### IEFBR14 — 空操作程序

```jcl
//STEP1 EXEC PGM=IEFBR14
//DD1 DD DSN=NEW.DATA,DISP=(NEW,CATLG,DELETE),
//    SPACE=(TRK,(10,5)),UNIT=SYSDA
```

**功能**：用于测试JCL、创建空数据集、分配空间。

### IEHLIST — 卷VTOC列表

```jcl
//LIST EXEC PGM=IEHLIST
//SYSPRINT DD SYSOUT=*
//VOL DD VOL=SER=123456,UNIT=3380
//SYSIN DD *
  LISTVTOC VOL=3380,SER=123456
/*
```

### IEBISAM — ISAM文件处理（遗留）

用于创建和处理ISAM数据集（已被VSAM取代）。

## 数据集列表和打印实用程序

### IEBPTPCH — 数据集打印/列表

```jcl
//PRINT EXEC PGM=IEBPTPCH
//SYSPRINT DD SYSOUT=*
//SYSUT1 DD DSN=INPUT.DATA,DISP=SHR
//SYSUT2 DD SYSOUT=*
//SYSIN DD *
  PRINT MAXFLDS=10
  RECORD FIELD=(80,1,1,CH)
/*
```

### IEBCOMPR — 数据集比较

```jcl
//CMP EXEC PGM=IEBCOMPR
//SYSPRINT DD SYSOUT=*
//SYSUT1 DD DSN=DATA1,DISP=SHR
//SYSUT2 DD DSN=DATA2,DISP=SHR
//SYSIN DD *
  COMPARE TYPORG=PS
/*
```

## 维护实用程序

### IEHPROGM — 数据集维护

```jcl
//MAINT EXEC PGM=IEHPROGM
//SYSPRINT DD SYSOUT=*
//DD1 DD VOL=SER=123456,UNIT=3380
//SYSIN DD *
  SCRATCH DSNAME=OLD.DATA,VOL=3380=123456
/*
```

**功能**：
- 擦除（scratch）数据集
- 重命名数据集
- 重建VTOC

## 实用程序参考表

| 实用程序 | 用途 | 主要DD语句 |
|---|---|---|
| `IEBGENER` | 复制顺序数据集 | SYSUT1, SYSUT2, SYSIN |
| `IEBCOPY` | 复制PDS/PDSE | INDD, OUTDD, SYSIN |
| `SORT` (DFSORT) | 排序/合并数据 | SORTIN, SORTOUT, SYSIN |
| `ICETOOL` | DFSORT多操作工具 | 输入/输出数据集, TOOLIN |
| `IEFBR14` | 空操作/空间分配 | 任何需要创建的DD |
| `IEBCOMPR` | 比较数据集 | SYSUT1, SYSUT2, SYSIN |
| `IEBPTPCH` | 打印数据集 | SYSUT1, SYSUT2, SYSIN |
| `IEHPROGM` | 数据集维护 | VOL, SYSIN |
| `IEHLIST` | 列出卷VTOC | VOL, SYSIN |

## JCL语句与实用程序的关系

JCL实用程序通常需要以下DD语句模式 ^[extracted]：

- `SYSPRINT` — 消息输出（通常SYSOUT=*）
- `SYSUT1/SYSUT2` — 输入/输出数据
- `SYSIN` — 控制语句

## 相关页面

- [[concepts/zos-io-data-management]] — VSAM和DFSMS
- [[concepts/zos-data-sets]] — 数据集和命名规则
- [[concepts/jcl-job-proc-step]] — JCL语句基础

## 来源

- IBM z/OS 3.1 MVS JCL User's Guide, SA23-1386-60, Chapter 2 and utility references