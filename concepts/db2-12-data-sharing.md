---
title: "DB2 12数据共享增强"
category: concepts
tags: [db2, data-sharing, peer-recovery, xa, lock-duplexing, zos]
sources: [下载/db2.pdf]
summary: DB2 12数据共享增强：自动对等恢复（Peer Recovery）无需ARM自动重启失败成员、XA全局事务支持跨成员路由、异步锁双工性能接近simplex模式。
provenance:
  extracted: 0.80
  inferred: 0.15
  ambiguous: 0.05
base_confidence: 0.65
lifecycle: draft
lifecycle_changed: 2026-05-14
created: 2026-05-14
updated: 2026-05-14
---

# DB2 12数据共享增强

## 概述

DB2 12对数据共享（Data Sharing）进行了多项关键增强，提升了可用性、可扩展性和性能。数据共享允许独立的DB2系统协同工作，提供高可用性和水平扩展能力。

## 自动对等恢复（Peer Recovery）

### 背景

传统的自动重启需要依赖**ARM（Automatic Restart Manager）**或外部机制。DB2 12引入了**对等恢复**功能，允许数据共享组中的成员自动重启失败的同伴成员，无需ARM或外部自动化。

### PEER_RECOVERY DSNZPARM

```sql
PEER_RECOVERY = NONE | RECOVER | ASSIST | BOTH
```

| 值 | 含义 |
|----|------|
| `NONE` | 该成员不参与对等恢复过程（默认） |
| `RECOVER` | 该成员在失败时由对等成员恢复 |
| `ASSIST` | 该成员负责协助恢复其他对等成员 |
| `BOTH` | `RECOVER`和`ASSIST`的组合 |

### 恢复流程

1. z/OS通知具有`PEER_RECOVERY=ASSIST`或`BOTH`设置的成员有对等成员失败
2. 所有幸存成员获取全局锁来串行化恢复过程
3. 第一个获得锁的协助成员使用`LIGHT(YES)`选项向失败成员发出`START DB2`命令
4. 使用失败成员上次使用的DSNZPARM加载模块自动重启
5. 如果轻量重启失败，另一个协助成员可以重试验证过程
6. 所有成员尝试后失败成员仍无法恢复时，需要手动干预

## XA支持全局事务

### 问题

在DB2 12之前，多个XA事务使用相同事务ID（XID）或全局事务的多个分支在数据共享环境中可能无法共享资源（锁），导致查询争用和时间outs。

### DB2 12解决方案

DB2 12改进了同一全局事务中多个连接跨数据共享组成员的服务方式：

1. **全局事务所有者确定**：第一个使用XA连接连接到成员的XA资源被称为全局事务的所有者
2. **SCA结构记录**：该成员在SCA结构中写入条目，记录全局事务ID、格式ID、所属成员的IP地址和端口
3. **后续连接路由**：后续XA资源通过另一成员连接时，该成员查询SCA结构确定所有者，构建DRDA连接请求并路由到所属成员

```java
// 示例：同一全局事务的两个分支连接到不同成员
xaR1.start(xid1, com.ibm.db2.jcc.DB2XAResource.TMLCS);
xaR2.start(xid2, com.ibm.db2.jcc.DB2XAResource.TMLCS);
// 两个UPDATE语句都在同一成员（所有者）上执行
s1.execute("UPDATE TEMP SET ID = 123456789 WHERE ID = 159357");
s2.execute("UPDATE TEMP SET ID = 789456123 WHERE ID = 753951");
```

## 改进的锁规避检查（Improved Lock Avoidance Checking）

### 背景

为获得最佳的锁规避或重用已删除空间，所有访问数据的应用程序应频繁发出COMMIT语句。但某些应用程序由于应用程序逻辑原因无法频繁COMMIT，导致系统级提交和读取LRSN保持较旧状态。

### DB2 12改进

DB2 12通过为提交LRSN和读取LRSN提供更大的粒度来改善性能和空间问题：

- 这些值保存在每个DB2成员的**对象级别（page set或partition）**内存中
- 每个成员最多可跟踪**500个对象级提交和读取LRSN值**
- 在数据共享环境中，对象级提交和读取LRSN值也存储在每个成员的**SCA结构**中

## 异步锁双工（Asynchronous Lock Duplexing）

### 同步双工的问题

传统的系统管理双工模式要求每次结构更新同时在主结构和次结构中执行，两个结构都完成才能返回请求。当两个结构之间距离较远（如超过10公里）时，同步方式会产生明显的性能开销。

### 异步双工解决方案

DB2 12引入**异步锁双工**，在保持高可用性优势的同时解决性能问题：

**工作原理：**
1. DB2事务的锁请求发送到IRLM和z/OS XES，仅发送到主锁结构
2. XES在主结构更新完成后立即向IRLM和DB2返回序列号
3. 同时，在后台向次结构发送更新请求
4. 每个线程跟踪次结构的更新序列号，最旧的保存在DB2成员级别
5. 当事务需要强制日志写入（如COMMIT）时，DB2和IRLM检查最旧序列号的请求是否已成功写入次结构

**性能优势：**
- 性能结果通常与simplex锁结构相似，因为主请求中无需等待双结构更新
- 连续异步后台更新确保次结构及时完成
- 在保持双工功能的同时实现良好的距离灵活性

**配置要求：**
- CF Level 21 (service level 02.16)
- z/OS V2.2 SPE with PTFs for APAR OA47796
- DB2 12 with PTFs for APAR PI66689
- IRLM 2.3 with PTFs for APAR PI68378

**启用方式：**
- CFRM couple data set (CDS) 必须使用新的`ASYNCDUPLEX`关键字格式化
- CFRM policy for DB2 lock structure必须使用新的`ASYNCONLY`关键字更新

## 相关概念

- [[references/ibm-db2-12-zos-technical-overview]] — 原始红皮书来源
- [[concepts/db2-12-scalability-availability]] — 可扩展性与高可用特性
- [[concepts/continuous-delivery]] — 连续交付与函数级别管理

## 来源

- IBM Redbooks SG24-8383-00, Chapter 5 "Data sharing", December 2016