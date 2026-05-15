---
title: "DB2 12时间维度表增强"
category: concepts
tags: [db2, temporal-tables, business-time, system-time, auditing, zos]
sources: [下载/db2.pdf]
summary: DB2 12时间维度表增强包括：应用期间包含/包含模型、参照约束支持、时态逻辑事务、审计能力（通过非确定性生成列实现）。
provenance:
  extracted: 0.82
  inferred: 0.13
  ambiguous: 0.05
base_confidence: 0.65
lifecycle: draft
lifecycle_changed: 2026-05-14
created: 2026-05-14
updated: 2026-05-14
---

# DB2 12时间维度表增强

## 概述

时间维度表（Temporal Tables）是DB2用于管理时间相关数据的核心特性，DB2 12在应用期间、参照约束、时态逻辑事务和审计能力等方面实现了重要增强。

## 两种时间周期类型

### 系统周期（System Period）

- 包含**系统周期时间表（System Temporal Table, STT）**
- 由系统维护的`SYSTEM_TIME`期间列对（`SYS_START`和`SYS_END`）
- 用于数据版本控制和历史追踪

### 应用周期（Application Period）

- 包含**应用周期时间表（Application Temporal Table, ATT）**
- 由应用程序维护的`BUSINESS_TIME`期间列对
- 允许应用程序控制数据的有效时间

## 应用期间增强

### 包含/包含模型（Inclusive-Inclusive）

DB2 12之前仅支持**包含/排他（inclusive-exclusive）**语义，即start time包含在周期内，end time排除在外。

DB2 12引入**包含/包含（inclusive-inclusive）**语义，start time和end time都包含在时间间隔内：

```sql
PERIOD BUSINESS_TIME (DSTART, DEND) INCLUSIVE
```

这使应用程序开发者有更大的灵活性来满足业务需求。

### 时态谓词变化

| 表达式 | Inclusive-Inclusive (DB2 12) | Inclusive-Exclusive (DB2 11) |
|--------|---------------------------|---------------------------|
| AS OF | `BUS_START <= expr AND BUS_END >= expr` | `BUS_START <= expr AND BUS_END > expr` |
| FROM...TO | `BUS_START < expr-2 AND BUS_END >= expr-1 AND expr-1 < expr-2` | `BUS_START < expr-2 AND BUS_END > expr-1 AND expr-1 < expr-2` |
| BETWEEN...AND | `BUS_START <= expr-2 AND BUS_END >= expr-1 AND expr-1 <= expr-2` | `BUS_START <= expr-2 AND BUS_END > expr-1 AND expr-1 <= expr-2` |

## 参照约束支持

DB2 12支持为包含`BUSINESS_TIME`期间的应用周期时间表定义**时态参照约束**。

### 实现要求

- 父表必须在`BUSINESS_TIME WITHOUT OVERLAPS`子句上定义唯一索引
- 子表必须具有对应外键的索引，带有`BUSINESS_TIME WITH OVERLAPS`子句
- 父表和子表的`BUSINESS_TIME`期间语义必须一致（都是inclusive-exclusive或都是inclusive-inclusive）

### 示例

```sql
CREATE TABLE myDEPT (
  DNO INTEGER NOT NULL,
  DNAME VARCHAR(30),
  DSTART DATE NOT NULL,
  DEND DATE NOT NULL,
  PERIOD BUSINESS_TIME (DSTART, DEND),
  PRIMARY KEY (DNO, BUSINESS_TIME WITHOUT OVERLAPS)
);

CREATE TABLE myEMP (
  ENO INTEGER NOT NULL,
  ESTART DATE NOT NULL,
  EEND DATE NOT NULL,
  EDEPTNO INTEGER,
  PERIOD BUSINESS_TIME (ESTART, EEND),
  PRIMARY KEY (ENO, BUSINESS_TIME WITHOUT OVERLAPS),
  FOREIGN KEY (EDEPTNO, PERIOD BUSINESS_TIME)
    REFERENCES myDEPT (DNO, PERIOD BUSINESS_TIME)
);
```

DB2会确保每个子行的时间周期都包含在父行的时间周期内（或一组连续的无间隙行）。

## 时态逻辑事务（Temporal Logical Transactions）

### 概念

传统的逻辑工作单元由COMMIT或ROLLBACK确定。DB2 12引入了**时态逻辑事务**，允许应用程序控制时间周期的生成时间，而不受COMMIT/ROLLBACK约束。

### 新的内置全局变量

| 变量 | 数据类型 | 说明 |
|------|---------|------|
| `SYSIBM.TEMPORAL_LOGICAL_TRANSACTION_TIME` | TIMESTAMP(12) | 控制SYSTEM_TIME期间的时间戳值；设为NULL（默认）时禁用时态逻辑事务 |
| `SYSIBM.TEMPORAL_LOGICAL_TRANSACTIONS` | SMALLINT | `0`（默认）：禁止单次提交范围内多个时态逻辑事务；`1`：允许 |

### 行为影响

- **INSERT/UPDATE of STT**：当前表的begin列值由DB2根据全局变量值生成
- **UPDATE/DELETE of STT**：历史表的end列值由DB2根据全局变量值生成
- **DELETE with ON DELETE ADD EXTRA ROW**：额外行的begin和end列值由DB2根据全局变量值生成

## 审计能力（Auditing Capabilities）

### 非确定性生成表达式列

DB2 12支持在表中定义**非确定性生成表达式列**，用于审计追踪：

```sql
USER_ID VARCHAR(128) GENERATED ALWAYS AS (SESSION_USER),
OPCODE CHAR(1) GENERATED ALWAYS AS (DATA CHANGE OPERATION)
```

### DATA CHANGE OPERATION支持的值

| 值 | 含义 |
|----|------|
| `'D'` | Delete操作 |
| `'I'` | Insert操作 |
| `'U'` | Update操作 |

### 支持的特殊寄存器和会话变量

**特殊寄存器：**
- `CURRENT CLIENT_ACCTG` → VARCHAR(255)
- `CURRENT CLIENT_APPLNAME` → VARCHAR(255)
- `CURRENT CLIENT_CORR_TOKEN` → VARCHAR(255)
- `CURRENT CLIENT_USERID` → VARCHAR(255)
- `CURRENT CLIENT_WRKSTNNAME` → VARCHAR(255)
- `CURRENT SERVER` → CHAR(16)
- `CURRENT SQLID` → VARCHAR(n) where n>=8
- `SESSION_USER` / `USER` → VARCHAR(128)

**会话变量：**
- `SYSIBM.PACKAGE_NAME` → VARCHAR(128)
- `SYSIBM.PACKAGE_SCHEMA` → VARCHAR(128)
- `SYSIBM.PACKAGE_VERSION` → VARCHAR(122)

### ON DELETE ADD EXTRA ROW子句

```sql
ALTER TABLE myPOLICIES
ADD VERSIONING USE HISTORY TABLE myPOLICIES_HIST
ON DELETE ADD EXTRA ROW;
```

此子句告诉DB2在从系统周期时间表（STT）删除行时，向关联的历史表插入额外行，用于追踪删除操作信息。

## 相关概念

- [[references/ibm-db2-12-zos-technical-overview]] — 原始红皮书来源
- [[concepts/db2-12-sql-enhancements]] — SQL增强特性
- [[concepts/db2-12-scalability-availability]] — 可扩展性与高可用特性

## 来源

- IBM Redbooks SG24-8383-00, Chapter 7 "Application enablement", December 2016