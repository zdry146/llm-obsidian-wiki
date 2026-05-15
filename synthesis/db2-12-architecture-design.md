---
title: "基于DB2 12特性的数据库架构设计"
category: synthesis
tags: [db2, architecture-design, system-design, zos, partition, high-availability]
sources: [下载/db2.pdf]
summary: 综合DB2 12核心特性（可扩展性、高可用、性能增强、时间维度表、数据共享）设计数据库架构的最佳实践与模式。
provenance:
  extracted: 0.55
  inferred: 0.35
  ambiguous: 0.10
base_confidence: 0.58
lifecycle: draft
lifecycle_changed: 2026-05-14
created: 2026-05-14
updated: 2026-05-14
---

# 基于DB2 12特性的数据库架构设计

## 概述

DB2 12引入了多项重大架构级特性，合理利用这些特性需要从顶层设计出发，而非局部优化。本文综合DB2 12的可扩展性、高可用、性能、时间维度表和数据共享等特性，提供架构设计指导。

**核心设计原则：**
- 充分利用PBR RPN分区结构实现水平扩展
- 通过时间维度表实现审计与业务分离
- 利用数据共享实现高可用与读写分离
- 在架构层面考虑性能而非仅依赖调优

## 一、分区架构设计

### 1.1 PBR RPN — 从"分区数诅咒"中解放

传统DB2分区表存在一个根本限制：**分区数与分区大小耦合**。增加分区数或修改分区边界都可能触发应用中断。

DB2 12的PBR RPN（Partition-By-Range with Relative Page Numbers）打破了这一限制：

| 维度 | 传统分区 | PBR RPN |
|------|---------|---------|
| 分区数 vs 大小 | 存在规则限制 | **解耦**，自由扩展 |
| RID结构 | 4字节 | **7字节**（2字节part + 5字节page）|
| 单表行数上限 | 受限 | **256万亿行** |
| 分区扩展 | 需重建 | **在线插入新分区** |

**设计建议：**
```
业务场景                    | 推荐分区策略
---------------------------|----------------------------------
超大规模流水数据            | PBR RPN + 月/周分区
需要频繁添加历史分区        | PBR RPN + 时间范围分区
单表超256万亿行需求         | PBR RPN（唯一可行方案）
```

### 1.2 分区键选择策略

分区键选择影响：
- **数据倾斜**：热点分区导致性能瓶颈
- **并行性**：跨分区查询的并行度
- **分区修剪**：查询能否有效裁剪分区

**原则：**
1. 选择高基数字段作为分区键
2. 避免时间戳类字段单独作为范围分区键（热点问题）
3. 考虑业务查询模式：频繁按时间+地区的查询，时间+地区复合分区可能更优
4. 对于MEMBER CLUSTER表，可考虑MEMBER作为分布维度

### 1.3 索引架构

**分区索引 vs 全局索引：**

| 索引类型 | 适用场景 | 维护成本 | 查询效率 |
|---------|---------|---------|---------|
| 分区索引 | 跨分区查询少、分区独立性强 | 低 | 分区内高效 |
| 全局索引 | 跨分区点查询、全局唯一性需求 | 高 | 全局最优 |

**FTB（Fast Traversal Block）优化：**
FTB是内存优化的索引结构，适用于缓存在buffer pool中的索引。设计时：
- 热点索引应保持在小内存 footprint 内以充分利用FTB
- 避免过大的索引（影响FTB缓存效率）

## 二、高可用架构

### 2.1 多层高可用设计

```
┌─────────────────────────────────────────────────────────┐
│                    应用层                                │
│  （连接池、负载均衡、故障切换）                            │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                  DB2 Data Sharing Group                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Member 1 │  │ Member 2 │  │ Member 3 │              │
│  │ (PRIMARY)│◄─┤(STANDBY) │  │(STANDBY) │              │
│  └──────────┘  └──────────┘  └──────────┘              │
│        ▲           ▲                                    │
│        │   Peer Recovery（自动对等恢复）                  │
│        └──────────────────────────────────────          │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                    IRLM + CF                             │
│          (Lock Structure + Lock Duplexing)              │
└─────────────────────────────────────────────────────────┘
```

### 2.2 对等恢复（Peer Recovery）设计

DB2 12的对等恢复允许数据共享组成员自动重启失败同伴，**无需ARM**。

**配置策略：**

```sql
PEER_RECOVERY = NONE | RECOVER | ASSIST | BOTH
```

| 成员角色 | 设置 | 说明 |
|---------|------|------|
| 普通成员 | `NONE` | 不参与对等恢复 |
| 备用成员 | `RECOVER` | 等待被恢复 |
| 协助成员 | `ASSIST` | 主动恢复他成员 |
| 关键成员 | `BOTH` | 双向恢复能力 |

**架构建议：**
- 至少2个成员配置为`ASSIST`或`BOTH`
- 避免所有成员配置为`RECOVER`（无人能救）
- 对等恢复不能替代完整的DR策略，它是**第一层自动响应**

### 2.3 异步锁双工（Async Lock Duplexing）

同步锁双工在跨数据中心场景下性能差（距离导致延迟）。DB2 12的异步锁双工解决了这一问题：

**工作原理：**
- 主锁结构更新完成后立即返回
- 后台异步更新次结构
- 通过序列号跟踪确保一致性

**架构建议：**
- 距离敏感型部署（跨城/跨数据中心）：**必须使用异步双工**
- 同机房低延迟场景：可考虑同步双工
- 异步双工性能接近simplex，但保持双活可用性

### 2.4 在线变更 — AREOR替代RBDP

**传统方式：** 索引压缩属性变更 → RBDP状态 → 索引不可用 → 应用中断风险

**DB2 12方式：** 索引压缩属性变更 → AREOR状态 → **应用继续访问** → 后台REORG完成

**设计模式：**
```
变更前：ALTER INDEX ix1 COMPRESS(YES)  →  立即生效但可能触发RBDP
DB2 12：ALTER INDEX ix1 COMPRESS(YES)  →  AREOR + 在线重组织
```

关键：不要害怕变更。AREOR使在线变更成为常态而非例外。

## 三、性能架构

### 3.1 插入性能架构

**快速插入算法（Fast Insert Algorithm）** 可实现**1100万次/秒**的非聚簇插入。

**启用条件：**
- Table space类型：Universal table space
- 集群方式：`MEMBER CLUSTER`
- 启用方式：系统级ZPARM或单表space级DDL

**架构设计：**

```sql
-- 推荐：启用快速插入的表空间
CREATE TABLESPACE ts_orders
  USING MEMBER CLUSTER
  INSERTALG(2)  -- 快速插入算法
  SEGSIZE(64);
```

**适用场景：**
- 高吞吐量写入（日志、事件、IoT数据）
- 非聚簇插入为主的ETL流程
- 需要减少redo日志的业务

### 3.2 查询性能架构

**UNION ALL 和 Outer Join 优化：**

DB2 12通过以下技术减少workfile物化：
- 列裁剪（不物化无用列）
- 谓词下推（减少中间结果）
- 重排join顺序（避免大结果集物化）
- ordering/fetch下推（减少排序开销）

**架构建议：**
- 对于bi-temporal表（透明归档），优先使用UNION ALL分表而非大单表
- 外连接查询注意表顺序（DB2 12会自动重排，但人工干预仍有效）
- 大数据集合的JOIN先过滤后关联

### 3.3 缓存架构

**FTB（Fast Traversal Block）** 设计考量：

| 因素 | 建议 |
|------|------|
| 索引总大小 | 保持可完全缓存在BP中 |
| 索引设计 | 避免过宽的索引列 |
| 查询模式 | 热点查询的索引优先FTB优化 |

## 四、时间维度表架构

### 4.1 双周期表设计

DB2 12支持在同一张表上同时使用**系统周期（审计）和应用周期（业务时间）**：

```sql
CREATE TABLE policy (
  policy_id BIGINT,
  coverage DECIMAL(15,2),
  -- 应用周期（业务时间）
  eff_date DATE,
  exp_date DATE,
  PERIOD BUSINESS_TIME (eff_date, exp_date),
  -- 系统周期（审计）
  sys_start TIMESTAMP(12),
  sys_end TIMESTAMP(12),
  PERIOD SYSTEM_TIME (sys_start, sys_end)
);
```

**架构模式：**

| 需求 | 推荐方案 |
|------|---------|
| 业务时间有效性 | BUSINESS_TIME（应用控制）|
| 审计历史追踪 | SYSTEM_TIME（系统控制）|
| 合规审计（谁、何时、什么）| SYSTEM_TIME + 生成列 |
| 业务时间点查询 | BUSINESS_TIME |
| 法律保留（7年）| SYSTEM_TIME + archive |

### 4.2 时态参照完整性

DB2 12支持跨表的时态外键约束：

```
                    ┌──────────────────┐
                    │    myDEPT        │
                    │ (Period: BUS_T)  │
                    │  PRIMARY KEY     │
                    │  + UNIQUE INDEX  │
                    └────────┬─────────┘
                             │ temporal FK
                             ▼
                    ┌──────────────────┐
                    │    myEMP         │
                    │ (Period: BUS_T)  │
                    │  FOREIGN KEY     │
                    │  WITH OVERLAPS   │
                    └──────────────────┘
```

**设计原则：**
- 父表：唯一索引带`WITHOUT OVERLAPS`
- 子表：外键带`WITH OVERLAPS`（允许重叠时间周期）
- 语义一致性：父表子表必须同为inclusive-inclusive或inclusive-exclusive

### 4.3 审计架构

**非确定性生成列**实现自动审计：

```sql
CREATE TABLE audit_sales (
  sale_id BIGINT,
  amount DECIMAL(15,2),
  -- 审计生成列
  user_id VARCHAR(128) GENERATED ALWAYS AS (SESSION_USER),
  app_name VARCHAR(255) GENERATED ALWAYS AS (CURRENT CLIENT_APPLNAME),
  op_code CHAR(1) GENERATED ALWAYS AS (DATA CHANGE OPERATION),
  -- 时间维度
  bus_start DATE,
  bus_end DATE,
  PERIOD BUSINESS_TIME (bus_start, bus_end)
);
```

**设计建议：**
- 审计表与业务表分离，通过外键关联
- 审计表的PK应包含业务键+时间周期
- 使用`ON DELETE ADD EXTRA ROW`保留删除历史

## 五、数据共享架构

### 5.1 读写分离架构

```
                 ┌─────────────────────────────┐
                 │     Write Traffic           │
                 │   (ALL Members Possible)    │
                 └──────────┬──────────────────┘
                            │
                 ┌──────────▼──────────────────┐
                 │   Data Sharing Group        │
                 │                             │
  ┌──────────────▼──────────────┐             │
  │       Member 1 (Writer)      │◄────────────┘
  └──────────────┬──────────────┘
                 │ 锁传播 / 全局事务
  ┌──────────────▼──────────────┐
  │       Member 2 (Reader)      │
  │  (同一全局事务的不同连接)     │
  └─────────────────────────────┘
```

**XA全局事务优化：**
- DB2 12自动路由同一XID的请求到**事务所有者成员**
- 避免了跨成员的锁争用
- 架构设计：XA应用应连接到**事务发起成员**

### 5.2 成员分布设计

| 场景 | 成员策略 |
|------|---------|
| 高可用为主 | 2-4成员，分布在不同LPAR/CPC |
| 读写分离 | 1主+多读，连接池路由 |
| 计算密集 | 多成员分担复杂查询 |
| 地理分布 | 异步锁双工+远程成员 |

### 5.3 锁优化设计

**对象级LRSN优化：**

DB2 12支持page set级别的提交LRSN和读取LRSN，每个成员最多跟踪500个对象级值。

**设计建议：**
- 频繁访问的热点表受益于对象级LRSN
- 避免大事务长时间持有锁（阻止LRSN推进）
- 应用程序应**频繁COMMIT**以释放LRSN压力

## 六、安全架构

### 6.1 最小权限原则

DB2 12支持更细粒度的权限：
- **UNLOAD权限**：不同于SELECT，可独立授权
- **CREATE/ALTER ON TABLESPACE**：表空间级权限
- **SETSESSIONUSER**：会话用户切换

### 6.2 所有权转移

```sql
TRANSFER OWNERSHIP OF TABLE myschema.mytable TO USER newowner;
```

**适用场景：**
- 应用程序退役后的数据迁移
- 所有权的组织调整
- 归档数据的管理权转移

## 七、综合架构模式

### 7.1 物联网（IoT）数据架构

```
┌─────────────────────────────────────────────────────────┐
│                    写入路径                              │
│  ┌─────────┐    ┌──────────────┐    ┌───────────────┐   │
│  │ 设备    │───▶│ MEMBER CLUSTER │───▶│ Fast Insert   │   │
│  │ 数据    │    │ Universal TS   │    │ (1100万/秒)   │   │
│  └─────────┘    │ PBR RPN       │    └───────────────┘   │
│                 │ + 时间分区    │                        │
│                 └──────────────┘                        │
└─────────────────────────────────────────────────────────┘
```

### 7.2 金融交易审计架构

```
┌─────────────────────────────────────────────────────────┐
│                   业务表（双周期）                        │
│  policy_id + BUSINESS_TIME + SYSTEM_TIME + 审计生成列    │
└─────────────────────────────────────────────────────────┘
         │                                    ▲
         │ REFERENTIAL CONSTRAINT             │
         │ (Temporal FK with OVERLAPS)        │
         ▼                                    │
┌─────────────────────────────────────────┐  │
│                  审计表                    │──┼── ON DELETE ADD EXTRA ROW
└─────────────────────────────────────────┘  │
         │                                    │
         │ SYSTEM_TIME ──────────────────────┘
         ▼
┌─────────────────────────────────────────┐
│               历史表                      │
│   (自动维护，保留完整变更历史)              │
└─────────────────────────────────────────┘
```

### 7.3 多租户SaaS架构

| 需求 | DB2 12方案 |
|------|-----------|
| 租户数据隔离 | 独立表空间 + 权限控制 |
| 租户规模扩展 | PBR RPN单表256万亿行 |
| 租户独立备份恢复 | SCOPE UPDATED关键字 |
| 跨租户分析 | UNION ALL视图 |
| 租户配额管理 | Resource limit facility |

## 八、迁移策略

### 8.1 函数级别规划

```
V12R1M100 (初始) → 验证兼容性
       ↓
V12R1M500 (最低新功能) → 不可回退
       ↓
V12R1M5xx (逐步激活) → 按需启用新功能
```

**策略：**
1. 新部署：从V12R1M500开始
2. 迁移：先在V12R1M100验证，确认后再升级
3. 激进策略：业务允许时可跳过中间函数级别
4. 保守策略：使用星函数级别回退

### 8.2 应用兼容性策略

- 所有新包：`APPLCOMPAT(V12R1M500)`
- 存量包：保持当前兼容性级别，按需升级
- 测试策略：回归测试 + SQL兼容性验证

## 相关页面

- [[references/ibm-db2-12-zos-technical-overview]] — 原始红皮书
- [[concepts/db2-12-scalability-availability]] — 可扩展性与高可用
- [[concepts/db2-12-performance-enhancements]] — 性能增强
- [[concepts/db2-12-sql-enhancements]] — SQL增强
- [[concepts/db2-12-temporal-tables]] — 时间维度表
- [[concepts/db2-12-data-sharing]] — 数据共享
- [[concepts/continuous-delivery]] — 连续交付与函数级别

## 来源

- IBM Redbooks SG24-8383-00, 综合DB2 12各章节架构设计考量