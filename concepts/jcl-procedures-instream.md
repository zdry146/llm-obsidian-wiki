---
title: JCL Procedures and In-Stream Processing
category: concepts
tags: [ibm, zos, jcl, procedure, instream, cataloged, PROC, PEND, JCLLIB]
sources: [下载/jcl user guide.pdf]
summary: In-stream过程（嵌入作业中）和cataloged过程（存储在PDS/PDSE中）的创建、调用和参数覆盖，以及INCLUDE组的使用。
provenance:
  extracted: 0.85
  inferred: 0.10
  ambiguous: 0.05
base_confidence: 0.72
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15T01:35:00Z
updated: 2026-05-15T01:35:00Z
---

# JCL Procedures and In-Stream Processing

## 概述

JCL支持两种类型的过程（procedures）来复用JCL代码 ^[extracted]：

1. **In-stream过程** — 嵌入在作业中的命名JCL语句集
2. **Cataloged过程** — 存储在分区数据集（PDS）或PDSE中的命名JCL语句集

## In-Stream过程

### 基本语法

```jcl
//PTEST PROC
//PSTA EXEC PGM=CALC
//DDA DD DSNAME=D.E.F,DISP=OLD
//DDB DD DSNAME=DATA1,DISP=(MOD,PASS)
//DDOUT DD SYSOUT=*
//PSTB EXEC PGM=PRNT
//DDC DD DSNAME=*.PSTA.DDB,DISP=OLD
//DDREP DD SYSOUT=A
// PEND
```

### 关键特性

- In-stream过程以 `PROC` 语句开始
- 以 `PEND` 语句结束
- 最多可在单个作业中编码15个in-stream过程 ^[extracted]
- 过程可以包含多个步骤

### 调用In-Stream过程

```jcl
//STEP1 EXEC PROC=PTEST
```

### 添加In-Stream数据到过程

```jcl
//STEP1 EXEC PROC=PTEST
//PSTA.IN DD *
.
(data)
.
/*
```

## Cataloged过程（Cataloged Procedures）

### 基本语法

```jcl
//MYPROC PROC
//MY1 EXEC PGM=WORK1
//MYDDA DD SYSOUT=A
//MYDDB DD SYSOUT=*
//MY2 EXEC PGM=TEXT5
//MYDDC DD DSNAME=F.G.H,DISP=OLD
//MYDDE DD SYSOUT=*
// PEND
```

### 存储位置

Cataloged过程存储在过程库（procedure library）中 ^[extracted]：

- **系统过程库**：`SYS1.PROCLIB`
- **用户指定的过程库**：通过 `JCLLIB` 语句指定

### 调用Cataloged过程

```jcl
//JOB2 JOB ,'JACKIE DIGIAN'
//STEPA EXEC PROC=MYPROC
```

### 过程库搜索顺序

```jcl
//IDLIB JCLLIB ORDER=(PRILIB.INCL.ONE,PRILIB.INC.TWO)
```

系统按 `ORDER` 参数指定的顺序搜索过程库，然后搜索 `SYS1.PROCLIB` 和安装定义的库 ^[extracted]。

## 符号参数（Symbolic Parameters）

### 定义默认值

```jcl
//MYPROC PROC DSN=USER.DATA
//STEP1 EXEC PGM=PROGRAMM
//DD1 DD DSN=&DSN
```

`PROC` 语句定义符号参数的默认值 ^[extracted]。

### 覆盖符号参数

```jcl
//STEP1 EXEC PROC=MYPROC,DSN=NEW.DATA
```

### SET语句

SET语句在作业中定义或更改变量 ^[extracted]：

```jcl
//SET1 SET DSN=NEW.DATA
```

## INCLUDE组

INCLUDE语句将PDS或PDSE成员中的JCL语句组嵌入到作业流中 ^[extracted]：

```jcl
//INOUT INCLUDE MEMBER=INOUTDD
```

### INCLUDE组示例

```jcl
//INOUT4 DD
DSNAME=DS4,UNIT=3380,VOL=SER=111112,
DISP=(NEW,KEEP),SPACE=(TRK,(5,1,2))
//INOUT5 DD
DSNAME=DS5,UNIT=3380,VOL=SER=111113,DISP=SHR
```

## 过程嵌套

Cataloged过程可以包含对其他cataloged过程的调用 ^[inferred]：

```jcl
//JOB1 JOB
//STEPA EXEC PROC=PROC1
//PROC1 PROC
//STEPB EXEC PROC=PROC2
//...
```

## 过程与JCL语句的关系

### 过程vs直接JCL

| 特性 | 过程 | 直接JCL |
|---|---|---|
| 复用性 | 可被多个作业调用 | 仅在当前作业中使用 |
| 维护 | 集中管理，易于更新 | 每个作业独立 |
| 参数化 | 支持符号参数覆盖 | 硬编码 |
| 测试 | 先作为in-stream测试，再编目 | 每个作业单独测试 |

### 测试过程

建议在将过程编目之前，先作为in-stream过程测试 ^[extracted]：

1. 先在作业中作为in-stream过程运行
2. 验证正确后，将PROC和PEND语句添加到过程代码
3. 将过程存储到过程库中
4. 使用 `EXEC PROC=name` 调用

## 过程中的COND参数

### 在调用时传递COND参数

```jcl
//TEST EXEC PROC=PROC4,COND.STEP4=((7,LT,STEP1),(5,EQ),EVEN)
```

此示例将COND参数传递给过程的STEP4步骤 ^[extracted]。

### 覆盖过程中的COND参数

```jcl
//TEST EXEC PROC=MYPROC,COND=((7,LT,STEP1),(5,EQ))
```

此EXE语句为调用过程的所有步骤建立COND参数，覆盖过程中编码的任何COND参数。

## 相关页面

- [[concepts/zos-jcl-basics]] — JCL基础和PROC语句
- [[concepts/jcl-job-proc-step]] — EXEC语句详解
- [[concepts/jcl-data-set-disposition]] — 数据集DISP和后向引用

## 来源

- IBM z/OS 3.1 MVS JCL User's Guide, SA23-1386-60, Chapter 2 "More complex jobs"