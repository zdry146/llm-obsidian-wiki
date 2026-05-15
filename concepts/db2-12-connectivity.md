---
title: "DB2 12连接性与管理例程"
category: concepts
tags: [db2, connectivity, drda, session-token, stored-procedures, zos]
sources: [下载/db2.pdf]
summary: DB2 12连接性增强：会话令牌（Session Token）支持、ROLLBACK后保留预准备动态语句、DRDA快速加载、profile监控增强。
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

# DB2 12连接性与管理例程

## 概述

DB2 12对分布式连接和管理例程进行了重要增强，包括会话令牌支持、动态语句持久化和Profile监控改进。

## 会话数据维护（Session Token Support）

### 背景

在DB2 11及之前，使用sysplex workload balancing (WLB)的客户端驱动程序必须**记住并追踪会话信息**（全局变量、客户端信息、特殊寄存器），以支持事务池化或连接重路由。这导致：
- 回放数据量增长
- 安全风险
- 性能下降

### DB2 12改进

DB2 12引入会话令牌支持：
- 客户端传递一个小的会话令牌（仅几个字节）到DB2服务器
- DB2在数据共享组级别维护会话数据
- 客户端重新连接时只需传递会话令牌，会话数据自动保持

### 会话目录表

| 表名 | 说明 |
|------|------|
| `SYSIBM.SYSSESSION` | 主会话数据表 |
| `SYSIBM.SYSSESSION_EX` | 扩展会话数据 |
| `SYSIBM.SYSSESSION_STATUS` | 会话状态 |

### 配置

- 客户端`clientApplcompat`需设置为`'V12R1'`以启用会话令牌支持
- 需要新的客户端驱动程序级别

### 会话数据超时

```sql
-MODIFY DDF SESSIDLE(100)
```

设置会话数据超时值（分钟）。

### 断开连接行为

当应用程序关闭逻辑连接时：
1. 分布式线程（DBAT）被池化
2. 连接变为非活动状态
3. 会话信息被释放

## ROLLBACK后保留预准备动态语句

### DB2 11行为

在DB2 11中，`KEEPDYNAMIC(YES)`仅适用于`COMMIT`请求，`ROLLBACK`会解除预准备动态语句：

```sql
PREPARE STMTID FOR SELECT ...
EXECUTE STMTID USING :H1
ROLLBACK  -- 语句被解除准备
EXECUTE STMTID USING :H2  -- 失败
```

### DB2 12改进

DB2 12将`KEEPDYNAMIC(YES)`功能扩展到`ROLLBACK`：

```sql
-- 包绑定时指定 APPLCOMPAT(V12R1M500) 和 KEEPDYNAMIC(YES)
PREPARE STMTID FOR SELECT ...
EXECUTE STMTID USING :H1
ROLLBACK  -- 语句保持准备状态
EXECUTE STMTID USING :H2  -- 成功
```

**条件**：包必须绑定时指定`APPLCOMPAT(V12R1M500)`和`KEEPDYNAMIC(YES)`。

**优势**：
- DBAT也保持活动状态
- 下次重新执行语句可立即恢复，无需重新准备
- 适用于本地应用程序

## DRDA快速加载

### 概述

DB2 12增强客户端驱动程序以支持**远程加载**到DB2 for z/OS服务器。

### 工作原理

- DB2 CLI API和命令行处理器（CLP）修改为向加载过程连续流式传输数据块
- 提取数据块并传递给LOAD utility的任务100%可由zIIP卸载
- 显著减少分布式客户端加载数据的elapsed时间

### 可用性

此功能在激活新函数之前即可使用。

## Profile监控增强

### 自动启动Profile

新增子系统参数`PROFILE_AUTOSTART`：
- `YES`：子系统启动时自动启动profile监控
- `NO`（默认）：启动时不自动启动profile监控

**注意**：如果DB2以`ACCESS(MAINT)`或`LIGHT(YES)`选项启动，DB2将忽略此设置。

### 支持全局变量

Profile现在支持通过全局变量定义监控条件。

### 支持通配符

Profile监控支持更灵活的通配符匹配。

### 空闲线程增强

改进空闲线程处理和监控。

## DB2提供的存储过程

### ADMIN_COMMAND_DB2

`ADMIN_COMMAND_DB2`存储过程的`DB2_LVL`结果集列的数据类型从`CHAR(3)`改为`CHAR(6)`。

### GET_CONFIG

`SYSPROC.GET_CONFIG`存储过程的XML输出包含新信息：
- 数据共享组当前函数级别
- 数据共享组最高已激活函数级别
- 数据共享组最高可能函数级别
- "Data Sharing Group Mode"键被移除

### 迁移注意

运行`DSNTIJRT`作业`MODE(INSTALL)`将定义结果表并绑定相关DBRM。

## 相关概念

- [[references/ibm-db2-12-zos-technical-overview]] — 原始红皮书来源
- [[concepts/db2-12-data-sharing]] — 数据共享增强
- [[concepts/continuous-delivery]] — 连续交付与函数级别管理

## 来源

- IBM Redbooks SG24-8383-00, Chapter 8 "Connectivity and administration routines", December 2016