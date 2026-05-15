---
title: JCL Storage and Resource Control
category: concepts
tags: [ibm, zos, jcl, storage, region, library, resource]
sources: [下载/jcl user guide.pdf]
summary: JCL存储请求（REGION/ADDRSPC）、程序库控制（系统库/私有库）、以及WLM调度环境选择。
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

# JCL Storage and Resource Control

## 概述

JCL提供多种机制来控制作业可用的存储和资源 ^[extracted]。

## 存储类型

MVS提供两种类型的存储 ^[extracted]：

| 类型 | 名称 | 说明 |
|---|---|---|
| **虚拟存储** | Virtual Storage | 地址空间为2GB，包含用户区域 |
| **中央存储** | Central (Real) Storage | 处理器可直接访问的存储 |

### REGION参数

`REGION` 参数指定作业或步骤可用的虚拟存储量 ^[extracted]：

```jcl
//J28 JOB ,'F. GOLAZESKI',CLASS=D
//S1 EXEC PGM=PROGREAL,REGION=20K,ADDRSPC=REAL
```

### ADDRSPC参数

`ADDRSPC` 参数指定使用虚拟还是中央存储 ^[extracted]：

| 值 | 含义 |
|---|---|
| `VIRT` | 虚拟存储（默认） |
| `REAL` | 中央存储（不能分页） |

### 中央存储要求

某些程序必须将其所有页面保存在连续的中部存储中，不能分页 ^[extracted]：

- 修改通道程序而通道程序处于活动状态时
- 时间高度敏感的程序

仅应为此类程序请求中央存储。

### REGION大小考虑

虚拟存储的REGION参数建立两个值：
1. 上限边界，限制可变长度GETMAIN的区域大小
2. IBM或安装提供的例程设置的第二个限制值

## 程序库资源控制

### 系统库（System Libraries）

系统库存储系统提供的程序和过程 ^[extracted]：

| 库 | 用途 |
|---|---|
| `SYS1.LINKLIB` | 系统链接库 |
| `SYS1.PROCLIB` | 系统过程库 |
| `SYS1.COBLIB` | COBOL库 |
| `SYS1.PARMLIB` | 参数库 |

### 私有库（Private Libraries）

私有库通过 `STEPLIB` 或 `JOBLIB` DD语句指定 ^[extracted]：

```jcl
//STEPLIB DD DSN=MY.LOADLIB,DISP=SHR
//JOBLIB DD DSN=MY.LOADLIB,DISP=SHR
```

| 语句 | 搜索顺序 |
|---|---|
| `STEPLIB` | 仅在当前步骤中搜索 |
| `JOBLIB` | 在整个作业的所有步骤中搜索 |

### 临时库

```jcl
//DD1 DD DSNAME=&&TEMPLIB,DISP=(NEW,PASS),
//    SPACE=(CYL,(5,1)),UNIT=SYSDA
```

临时库在作业结束时删除。

## 调度环境（Scheduling Environments）

WLM调度环境定义作业执行所需的资源状态 ^[extracted]：

```jcl
//JOBA JOB 1,'STEVE HAMILTON',SCHENV=DB2LATE
```

调度环境与SYSAFF/SYSTEM参数不同：
- 调度环境是抽象的、动态的
- SYSAFF/SYSTEM是具体的、静态的

## 处理器选择

### JES2 SYSAFF

```jcl
/*JOBPARM SYSAFF=SYS2
```

### JES3 SYSTEM

```jcl
//*MAIN SYSTEM=(PRS1,PRS3)
```

## Spool分区（JES3）

```jcl
//*MAIN SPART=partition-name
```

作业的输入spool数据集分配给默认spool分区；sysout数据集始终放在其输出类的分区中。

## I/O到处理比率（JES3）

```jcl
//*MAIN IORATE=HIGH
//*MAIN IORATE=MED
//*MAIN IORATE=LOW
```

IORATE参数帮助JES3平衡处理器密集型和I/O密集型作业的混合。

## 相关页面

- [[concepts/zos-workload-management]] — WLM工作负载管理
- [[concepts/zos-batch-processing]] — 批处理和资源分配
- [[concepts/jcl-job-proc-step]] — EXEC语句和REGION参数

## 来源

- IBM z/OS 3.1 MVS JCL User's Guide, SA23-1386-60, Chapter 9 "Entering jobs - resource control"