---
title: JCL Conditional Execution
category: concepts
tags: [ibm, zos, jcl, conditional, if-then-else, cond, return-code]
sources: [下载/jcl user guide.pdf]
summary: JCL条件执行机制：IF/THEN/ELSE/ENDIF语句构造（基于返回码、abend条件）和COND参数，控制作业步的跳过或执行。
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

# JCL Conditional Execution

## 概述

JCL提供两种条件执行机制 ^[extracted]：

1. **IF/THEN/ELSE/ENDIF语句构造** — 基于返回码、abend条件或完成码有条件地执行作业步
2. **COND参数** — 在JOB或EXEC语句上根据先前步骤的结果跳过或执行步骤

IF/THEN/ELSE/ENDIF是更现代、功能更强的机制，推荐使用。

## IF/THEN/ELSE/ENDIF语句构造

### 基本语法

```jcl
//[name]  IF (relational expression)
    EXEC PGM=stepname      // THEN clause — 条件为真时执行
//[name]  ELSE
    EXEC PGM=stepname      // ELSE clause — 条件为假时执行
//[name]  ENDIF
```

### 关系表达式

关系表达式由以下元素组成 ^[extracted]：

- **比较运算符**: `=`, `>`, `<`, `>=`, `<=`, `<>` (不等于)
- **逻辑运算符**: `&` (AND), `|` (OR)
- **非运算符**: `¬` (NOT)
- **关系表达式关键字**

### 关系表达式关键字

| 关键字 | 含义 |
|---|---|
| `RC` | 返回码（return code） |
| `CANCEL` | 作业被取消 |
| `ABEND` | 作业异常终止 |
| `SYSTEM` | 系统abend完成码 |
| `USER` | 用户abend完成码 |

### 比较运算符

| 运算符 | 含义 |
|---|---|
| `EQ` | 等于 |
| `NE` | 不等于 |
| `GT` | 大于 |
| `LT` | 小于 |
| `GE` | 大于或等于 |
| `LE` | 小于或等于 |

### 语法示例

```jcl
//IFGOOD IF (RC = 0)
    EXEC PGM=CONTINUE
//IFBAD  ELSE
    EXEC PGM=ERRORRTN
//ENDC   ENDIF
```

## 条件执行示例

### 基于返回码的测试

```jcl
//RCTEST IF (STEP1.RC = 10)
//STEP3 EXEC PGM=PROCESS
//IFNOT  ELSE
//SKIP   EXEC PGM=SKIPIT
//ENDTEST ENDIF
```

如果STEP1返回码为10，执行STEP3；否则执行SKIP步骤。

### 兼容的返回码测试

某些IBM程序产生标准返回码 ^[extracted]：

| 返回码 | 含义 |
|---|---|
| 4 | 发现次要错误，但产生了编译程序或加载模块，执行可能成功 |
| 8 | 发现重大错误，但产生了编译程序或加载模块，执行可能不成功 |
| 12 | 发现严重错误，未产生编译程序或加载模块 |
| 16 | 发现不可恢复的错误 |

```jcl
//NOTBAD IF (RC > 4)
//BADERR EXEC PGM=ERRRTN
//NOGOOD ELSE
//NEXTSTEP EXEC PGM=CONTINUE
//ENDC   ENDIF
```

返回码大于4时执行错误处理程序。

### 嵌套IF语句

IF/THEN/ELSE/ENDIF语句构造最多可嵌套15层 ^[extracted]。

### 作业级和步骤级评估

**作业级评估**：如果不指定stepname，IF/THEN/ELSE/ENDIF语句构造评估前面所有步骤的返回码。

**步骤级评估**：要测试单个步骤，指定要测试的步骤名。测试过程步骤时，指定 `stepname.procstepname`。

### 与COND参数的关系

当同时指定IF/THEN/ELSE/ENDIF语句构造和COND参数时，只有两者都评估为执行时，作业步才会执行 ^[extracted]。

## COND参数

### JOB语句上的COND

```jcl
//MYJOB JOB ACCT,'NAME',COND=(10,LT)
```

`COND=(code,test)` — 如果指定条件的计算结果为真，则跳过整个作业。

### EXEC语句上的COND

```jcl
//STEP2 EXEC PGM=B,COND=(7,LT)
```

`COND=(code,test,stepname)` — 在步骤执行前评估条件，如果为真则跳过该步骤。

### COND测试类型

| 测试 | 含义 |
|---|---|
| `GT` | 大于 |
| `LT` | 小于 |
| `EQ` | 等于 |
| `NE` | 不等于 |
| `GE` | 大于或等于 |
| `LE` | 小于或等于 |
| `ONLY` | 仅在前一步异常终止时跳过 |
| `EVEN` | 仅在前一步产生非零返回码时跳过 |

### COND参数示例

```jcl
//STEP1 EXEC PGM=A
//STEP2 EXEC PGM=B,COND=(5,GT,STEP1)
//STEP3 EXEC PGM=C,COND=((8,GT,STEP1),(2,EQ))
```

## IF vs COND

**IF/THEN/ELSE/ENDIF的优势** ^[extracted]：

- 可以测试abend条件和完成码（而不仅仅是返回码）
- 支持ELSE子句（COND参数不支持）
- 支持嵌套
- 更清晰的逻辑表达

**何时使用COND**：与IF/THEN/ELSE/ENDIF同时指定时，两者都必须评估为执行才能运行步骤。

## 步骤执行后的异常终止

步骤异常终止通常导致系统跳过后续步骤并终止作业。但IF/THEN/ELSE/ENDIF允许请求在前一步异常终止时执行某一步骤 ^[extracted]。

## 相关页面

- [[concepts/zos-jcl-basics]] — JCL基础概述
- [[concepts/jcl-job-proc-step]] — EXEC语句和COND参数
- [[concepts/cics-exec-api]] — EXEC CICS API中的响应码机制（作为对比）

## 来源

- IBM z/OS 3.1 MVS JCL User's Guide, SA23-1386-60, Chapter 10 "Processing jobs - processing control"