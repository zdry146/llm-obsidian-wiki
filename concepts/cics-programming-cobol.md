---
title: CICS COBOL编程
category: concepts
tags: [cics, cobol, programming, commarea, pseudo-conversational, paybus, paypgm]
sources: [/home/openclaw/下载/CICS.pdf]
summary: CICS COBOL应用开发核心：展示逻辑（PAYPGM）与业务逻辑（PAYBUS）分离、COMMAREA状态管理、伪会话编程模式。
provenance:
  extracted: 0.80
  inferred: 0.15
  ambiguous: 0.05
base_confidence: 0.80
lifecycle: draft
lifecycle_changed: 2026-05-14
created: 2026-05-14
updated: 2026-05-14
---

# CICS COBOL编程

## 工资单应用架构

本书的示例应用分为两个程序 ^[extracted]：

- **PAYPGM** — 展示逻辑程序（Presentation Logic），负责与3270终端交互
- **PAYBUS** — 业务逻辑程序（Business Logic），负责实际的数据处理

两者通过**COMMAREA**通信 ^[extracted]。

## COMMAREA（通信区）

COMMAREA是CICS程序之间传递数据和维护伪会话状态的核心机制 ^[extracted]。

### COMMAREA的结构

```cobol
* PAYPGM的WORKING-STORAGE中定义:
01  COMMAREA.
    05 CA-REQUEST       PIC X(10).    * 4字节请求 + 1字节部门 + 5字节员工号
    05 CA-INDICATORS     PIC X(10).    * 状态标志（'n'或'y'）
    05 CA-DATA           PIC X(...).   * 记录数据
```

### COMMAREA的Linkage使用

```cobol
* PAYPGM的LINKAGE SECTION中:
LINKAGE SECTION.
01  DFHCOMMAREA.
    05 CA-REQUEST       PIC X(10).
    ...

* PAYBUS通过LINKAGE SECTION接收:
LINKAGE SECTION.
01  DFHCOMMAREA.
    05 CA-REQUEST       PIC X(10).
    ...
```

### 为什么COMMAREA同时出现在Working Storage和Linkage

- **首次执行**：PAYPGM在WORKING-STORAGE中创建COMMAREA，填充初始值
- **后续执行**：PAYPGM将LINKAGE中的数据移回WORKING-STORAGE处理，再放回LINKAGE传递 ^[extracted]

这是CICS程序的常见模式 ^[inferred]。

## 展示逻辑（PAYPGM）

### 首次执行检测

```cobol
IF EIBCALEN = 0
    * 这是首次执行，没有COMMAREA
    PERFORM FIRST-TIME-THRU
ELSE
    * 有COMMAREA，不是首次执行，恢复状态
    PERFORM CONTINUE-PROCESSING
END-IF
```

`EIBCALEN`是EIB中的字段，表示传入的COMMAREA长度。为0表示这是该程序的首次调用 ^[extracted]。

### 函数处理

函数由用户按键触发。EIB中的`EIBAID`字段包含用户按键信息 ^[extracted]：

```cobol
EVALUATE EIBAID
    WHEN DFHENTER   PERFORM DO-WORK
    WHEN DFHPF3     PERFORM DO-DISPLAY
    WHEN DFHPF4     PERFORM DO-ADD
    WHEN DFHPF5     PERFORM DO-UPDATE
    WHEN DFHPF6     PERFORM DO-DELETE
    ...
END-EVALUATE
```

### 字段验证

展示逻辑负责验证用户输入的字段 ^[extracted]：

- 检查字段长度（通过`RECEIVE MAP`返回的长度）
- 更新COMMAREA中的状态标志
- 调用业务逻辑前进行初步验证

### 调用业务逻辑

```cobol
EXEC CICS LINK PROGRAM('PAYBUS')
    COMMAREA(COMMAREA)
    LENGTH(LENGTH OF COMMAREA)
END-EXEC
```

LINK命令是同步的 — 调用PAYBUS，等待其完成，然后继续 ^[inferred]。

### 业务逻辑返回后的处理

```cobol
IF EIBRESP != DFHRESP(NORMAL)
    * LINK命令本身失败
    PERFORM HANDLE-LINK-ERROR
END-IF
IF CA-ERROR-FLAG = 'Y'
    * 业务逻辑返回了错误
    PERFORM HANDLE-BUSINESS-ERROR
END-IF
```

## 业务逻辑（PAYBUS）

### COMMAREA接收

PAYBUS通过LINKAGE SECTION接收COMMAREA ^[extracted]：

```cobol
LINKAGE SECTION.
01  DFHCOMMAREA.
    05 CA-REQUEST       PIC X(10).
    ...
PROCEDURE DIVISION USING DFHCOMMAREA.
```

### 伪会话状态重置

对于需要两步完成的操作（ADD、UPDATE、DELETE），如果用户在第一步后中断，标志需要重置 ^[extracted]：

```cobol
* 如果用户中断了ADD/UPDATE/DELETE操作，重置标志
IF CA-FUNCTION = 'ADD' OR 'UPDATE' OR 'DELETE'
    IF CA-STEP = '1'
        MOVE 'n' TO CA-FLAG-READY
    END-IF
END-IF
```

### 函数路由

```cobol
EVALUATE CA-REQUEST
    WHEN 'DISP'   PERFORM DO-DISPLAY
    WHEN 'ADD'    PERFORM DO-ADD
    WHEN 'UPDT'   PERFORM DO-UPDATE
    WHEN 'DEL'    PERFORM DO-DELETE
    WHEN 'FWDR'   PERFORM DO-FORWARD
    WHEN 'BACK'   PERFORM DO-BACKWARD
END-EVALUATE
```

### READ with UPDATE

更新记录的标准模式 ^[extracted]：

```cobol
EXEC CICS READ FILE('PAYROLL')
    RIDFLD(ws-key)
    INTO(payroll-record)
    UPDATE
END-EXEC
* ... 处理字段更新 ...
EXEC CICS REWRITE FILE('PAYROLL')
    FROM(payroll-record)
END-EXEC
```

READ with UPDATE锁定记录，确保其他用户无法同时更新 ^[extracted]。

### 浏览（Browsing）

浏览键控文件时，CICS提供STARTBR、READNEXT、READPREV命令 ^[extracted]：

```cobol
EXEC CICS STARTBR FILE('PAYROLL')
    RIDFLD(ws-key)
    GTEQ
END-EXEC

EXEC CICS READNEXT FILE('PAYROLL')
    RIDFLD(ws-key)
    INTO(payroll-record)
END-EXEC
```

注意：两次连续的READNEXT是设计如此 — 第一次定位到当前记录，第二次读取下一条记录 ^[extracted]。

## 伪会话编程的关键点

1. **每次交互后必须返回** — `EXEC CICS RETURN`，不能像普通程序那样"等待输入"
2. **状态必须持久化到COMMAREA** — 下次调用时能恢复执行位置
3. **两阶段操作需要标志** — ADD/UPDATE/DELETE需要两步，确认后才真正执行
4. **记录锁定** — 通过READ with UPDATE获取锁，异常时由CICS自动回滚

## 相关概念

- [[concepts/cics-application-development]] — 编程范式和资源定义
- [[concepts/cics-exec-api]] — EXEC CICS API详细说明
- [[concepts/cics-channels-containers]] — 从COMMAREA演进到Channels和Containers
- [[references/modernizing-applications-ibm-cics]] — 原始红皮书来源

## 来源

- IBM Redbooks REDP-5628-00, Chapter 4 "Programming an IBM CICS application in COBOL", December 2020