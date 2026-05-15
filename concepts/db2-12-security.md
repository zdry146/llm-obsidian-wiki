---
title: "DB2 12安全增强"
category: concepts
tags: [db2, security, ownership, privileges, racf, zos]
sources: [下载/db2.pdf]
summary: DB2 12安全增强：无需SYSADM的安装迁移、UNLOAD权限、TRANSFER OWNERSHIP SQL语句，以及与RACF的集成。
provenance:
  extracted: 0.75
  inferred: 0.20
  ambiguous: 0.05
base_confidence: 0.65
lifecycle: draft
lifecycle_changed: 2026-05-14
created: 2026-05-14
updated: 2026-05-14
---

# DB2 12安全增强

## 概述

DB2 12在安全方面进行了重要增强，降低了管理复杂度并提高了对象保护能力。

## 无需SYSADM的安装和迁移

### 背景

在DB2 11及之前版本，安装或迁移DB2需要SYSADM权限，这给安全管理带来挑战。

### DB2 12改进

DB2 12允许在**不需要SYSADM权限**的情况下安装或迁移DB2，简化了从任何地方、任何计算机通过基于Web的应用程序（z/OS Management Facility）进行迁移的过程。

### z/OSMF集成

DB2 12支持使用z/OSMF：
1. 使用DB2安装CLIST和面板生成z/OSMF artifacts
2. 将生成的artifacts提供给z/OSMF
3. 实现更灵活的安装和迁移流程

## UNLOAD权限

### 新增权限

DB2 12为表和视图新增了`UNLOAD`权限：

```sql
GRANT UNLOAD ON TABLE myTable TO user1;
REVOKE UNLOAD ON TABLE myTable FROM user1;
```

### 执行方式

可通过以下方式执行：
- **DB2安全设施**：直接通过DB2管理
- **RACF**：通过资源访问控制设施管理

### 权限要求

在没有`DATAACCESS`或`SYSADM`权限的情况下，`UNLOAD`权限允许用户使用`UNLOAD`工具从表中卸载数据。

## TRANSFER OWNERSHIP SQL语句

### 概述

DB2 12引入`TRANSFER OWNERSHIP` SQL语句，支持在保持对象可用的情况下更改对象所有权，无需删除和重新创建对象。

### 语法

```sql
TRANSFER OWNERSHIP OF table myTable TO newOwner;
```

### 支持的对象

- 表
- 视图
- 其他数据库对象

### 行为

- 变更后对象立即可用
- 之前依赖旧所有者的权限仍然有效
- 新所有者获得对象的所有权限

### 与DB2 11对比

| 特性 | DB2 11 | DB2 12 |
|------|--------|--------|
| 所有权变更 | 必须DROP和RECREATE对象 | TRANSFER OWNERSHIP直接变更 |
| 可用性 | 变更期间对象不可用 | 对象持续可用 |
| 依赖对象 | 需要重新绑定相关包 | 自动处理 |

## 对象所有权变更目录追踪

### IFCID追踪

DB2 12通过IFCID记录所有权变更：

- IFCID 002：RDS统计块（包含新字段）
- IFCID 062：语句类型
- IFCID 140：源对象所有者和名称
- IFCID 361：源对象所有者和名称

### 字段

| 字段 | 说明 |
|------|------|
| `SOURCE_OBJECT_OWNER` | 当前对象所有者 |
| `SOURCE_OBJECT_NAME` | 源对象名称 |

## 使用RACF进行安全控制

### DB2安全设施 vs RACF

| 方式 | 说明 |
|------|------|
| DB2安全设施 | DB2原生安全管理 |
| RACF | z/OS资源访问控制 |

### 授权决策

DB2通过以下层次做出授权决策：
1. DB2内部检查
2. 外部安全管理器（如RACF）
3. 综合权限验证

### 与z/OS系统集成

DB2 12安全增强与z/OS安全基础设施深度集成，提供企业级安全保护。

## 相关概念

- [[references/ibm-db2-12-zos-technical-overview]] — 原始红皮书来源
- [[concepts/db2-12-scalability-availability]] — 可扩展性与高可用特性
- [[concepts/continuous-delivery]] — 连续交付与函数级别管理

## 来源

- IBM Redbooks SG24-8383-00, Chapter 10 "Security", December 2016