---
title: JCL Scheduling and Timing
category: concepts
tags: [ibm, zos, jcl, time, scheduling, jes2, jes3, deadline, periodic]
sources: [下载/jcl user guide.pdf]
summary: TIME参数控制作业/CPU时间，JES2/JES3调度参数（SYSAFF/SYSTEM），deadline/periodic调度，以及处理器选择机制。
provenance:
  extracted: 0.82
  inferred: 0.12
  ambiguous: 0.06
base_confidence: 0.71
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15T01:35:00Z
updated: 2026-05-15T01:35:00Z
---

# JCL Scheduling and Timing

## TIME参数

TIME参数限制作业或步骤的CPU时间 ^[extracted]：

```jcl
//JOB1 JOB ,'NAME',TIME=(5,30)  5分30秒
//STEP1 EXEC PGM=PGM,TIME=2     2分钟
```

### TIME参数格式

| 格式 | 含义 | 示例 |
|---|---|---|
| `TIME=mmm` | 分钟 | `TIME=5` = 5分钟 |
| `TIME=(mmm,ss)` | 分钟:秒 | `TIME=(5,30)` = 5分30秒 |
| `TIME=1440` | 无限制（仅JOB） | `TIME=1440` |
| `TIME=NOLIMIT` | 无限制（仅JOB） | |

### JOB vs EXEC TIME

- **JOB TIME** — 限制整个作业的累计CPU时间
- **EXEC TIME** — 限制特定步骤的CPU时间

如果REGION参数请求central存储（ADDRSPC=REAL），TIME参数的值必须足够大以包含程序执行和所有GETMAIN请求的空间 ^[extracted]。

## JES2时间参数

### /*JOBPARM TIME

```jcl
/*JOBPARM TIME=30
```

限制作业的执行时间（分钟）。

## Deadline和Periodic调度（JES3）

### Deadline调度

deadline调度在指定时间之前启动作业 ^[extracted]：

```jcl
//*MAIN DEADLINE=0900
```

作业将在早上9:00之前开始执行。如果无法在指定时间前启动，作业将根据优先级等待。

### Periodic调度

periodic调度定期启动作业 ^[extracted]：

```jcl
//*MAIN PERIODIC=120
```

作业每120分钟执行一次。

## 处理器选择

### WLM调度环境（SCHENV）

SCHENV参数指定WLM调度环境名称 ^[extracted]：

```jcl
//JOBA JOB 1,'STEVE HAMILTON',SCHENV=DB2LATE
```

调度环境是资源和所需设置列表。通过将调度环境名称与作业关联，确保作业仅在满足资源状态要求的系统上调度执行。

### JES2 SYSAFF参数

在JES2多访问spool配置中，可以请求在特定系统上执行 ^[extracted]：

```jcl
/*JOBPARM SYSAFF=SYS2
/*JOBPARM SYSAFF=(S333,IND)
/*JOBPARM SYSAFF=*
/*JOBPARM SYSAFF=ANY
```

| 值 | 含义 |
|---|---|
| `SYSAFF=cccc` | 指定系统处理作业 |
| `SYSAFF=(c1,c2,c3)` | 多个指定系统 |
| `SYSAFF=*` | 当前系统 |
| `SYSAFF=ANY` | 任何系统 |
| `IND` | 独立模式（用于测试新组件） |

### JES3 SYSTEM参数

JES3根据作业需要的资源自动选择处理器 ^[extracted]：

```jcl
//*MAIN SYSTEM=ANY
//*MAIN SYSTEM=JGLOBAL
//*MAIN SYSTEM=(PRS1,PRS3)
```

| 值 | 含义 |
|---|---|
| `SYSTEM=ANY` | 任何可用系统 |
| `SYSTEM=JGLOBAL` | 全局处理器 |
| `SYSTEM=JLOCAL` | 本地处理器 |
| `SYSTEM=(name1,name2)` | 指定处理器列表 |
| `SYSTEM=/(name1,name2)` | 排除指定处理器 |

## 性能控制

### 按作业类分配（JES3）

```jcl
//*MAIN CLASS=IMSBATCH
```

作业类定义作业的优先级和可用的系统资源。

### 选择优先级（JES2/JES3）

```jcl
//JOB10 JOB ,'NAME',PRTY=14
/*PRIORITY 14
```

### 优先级老化

JES2根据作业等待执行的时间增加作业优先级 ^[extracted]。

JES3根据作业被跳过选择的次数增加优先级（可能因为设备不足、所需卷或数据集不可用，或存储不足）。

### I/O与处理比率（JES3）

```jcl
//*MAIN IORATE=HIGH
//*MAIN IORATE=LOW
//*MAIN IORATE=MED
```

IORATE参数指示作业包含的I/O指令与处理指令相比是低、中还是高。JES3使用此值来平衡处理器密集型处理和I/O密集型处理。

## Spool分区（JES3）

JES3将spool卷划分为分区（partitions），可在作业级别覆盖分配 ^[extracted]：

```jcl
//*MAIN SPART=partition-name
```

sysout数据集始终放在其输出类的分区中。

## 相关页面

- [[concepts/zos-workload-management]] — WLM工作负载管理
- [[concepts/zos-batch-processing]] — 批处理和作业调度
- [[concepts/jcl-job-proc-step]] — TIME参数在EXEC语句中的使用

## 来源

- IBM z/OS 3.1 MVS JCL User's Guide, SA23-1386-60, Chapters 5, 10-11