---
title: CICS DevOps
category: concepts
tags: [cics, devops, jenkins, zowe, galasa, ibm-z, ci-cd, pipeline]
sources: [/home/openclaw/下载/CICS.pdf]
summary: IBM Z上的DevOps实践：Zowe CLI连接CICS、IBM Z Open Editor开发COBOL/Java、Galasa集成测试、UrbanCode Deploy自动化部署。
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

# CICS DevOps

## DevOps概述

DevOps是敏捷和精益原则的结合，强调持续改进 ^[extracted]。核心是CI/CD（持续集成/持续交付）：

- **持续集成（CI）** — 代码频繁合并到共享主干，通过自动化构建和测试验证
- **持续交付（CD）** — 自动化部署管道，将软件交付到各种环境

IBM的DevOps方法强调**企业文化** — 开发和运维之间的协作比工具更重要 ^[extracted]。

## IBM Z上的CICS DevOps

### 核心理念

DevOps对目标平台是透明的 — z/OS和CICS开发完全可以融入开放的分布式管道 ^[extracted]。

IBM提供跨平台的交付管道，整合开源和商业工具 ^[extracted]：

```
Git/Jira ──> Jenkins ──> SonarQube ──> UrbanCode Deploy ──> z/OS/CICS
```

## 工具链

### 开发环境

#### IBM Z Open Editor（Visual Studio Code扩展）

提供COBOL和CICS命令的自动补全 ^[extracted]：

- 输入`EXEC CICS`后自动提示可用命令
- 按Ctrl+Space可查看命令的所有属性

#### Zowe CLI

从命令行连接CICS Region ^[extracted]：

```bash
zowe cics --help
zowe cics get program PAYPGM
zowe cics refresh program PAYPGM
```

**关键命令：**
- `get` — 查看CICS资源（如程序）定义
- `refresh`（相当于3270界面的NEWCOPY/PHASEIN）— 强制CICS加载新版本的程序 ^[extracted]

**Zowe配置文件：** 需要协议（HTTP/HTTPS）、主机、端口、用户名、密码和Region名称 ^[extracted]。

### 源码管理

Git是事实标准 ^[extracted]。IBM Z Open Editor和Zowe CLI都支持Git集成。CICS应用（无论语言）可以完全参与Git生态 ^[extracted]。

**建议：** 全企业使用单一源码管理系统（RTC或Git），保持单一真相源 ^[extracted]。

### 构建

| 语言 | 构建工具 |
|---|---|
| Java | Maven或Gradle |
| COBOL/PL/I | IBM Dependency Based Build（DBB）+ Groovy自动化 |

CICS TS V5.6+ 支持Maven Central上的CICS Java库，简化CI/CD管道构建 ^[extracted]。

### 流水线自动化

Jenkins是最常用的CI/CD协调器 ^[extracted]。它提供：
- 单一审计点
- 与开源工具的集成
- 统一的流水线视图

### 测试

#### 单元测试

IBM Z Open Unit Test提供CICS程序的自动化单元测试能力 ^[extracted]：

- 支持CICS程序的桩（Stub）能力
- 无需部署到CICS即可进行单元测试
- 支持自动化数据捕获和录制

#### 集成测试：Galasa

Galasa是开源测试自动化框架，源于CICS开发团队的DevOps实践 ^[extracted]。

**特点：**
- 支持3270脚本、Web自动化（Selenium）等多种测试方式
- 测试可以在本地笔记本运行，但连接 mainframe
- 与Jenkins管道集成，基于变更集动态选择测试集

**Galasa的深度IBM Z集成：** 测试可以在同一环境中覆盖CICS、DB2、MQ等组件 ^[inferred]。

### 部署

IBM UrbanCode Deploy是自动化部署工具，支持从测试到生产的自动化应用部署 ^[extracted]。

**CICS TS插件功能：**
- 安装/卸载CICS资源
- 管道扫描
- 资源开放/关闭
- 以及更多操作

UrbanCode Deploy提供审计追踪、版本控制和审批流程，对生产环境尤其重要 ^[extracted]。

### 分析

IBM Application Discovery可视化应用依赖关系，自动化发现工作配合源码管理集成 ^[extracted]。

与CICS Interdependency Analyzer结合，可以提供静态代码分析和完整运行时分析的360度视图 ^[extracted]。

## 完整的DevOps管道

```
源代码 ──> 构建 ──> 单元测试 ──> 静态分析 ──> 集成测试 ──> 部署 ──> 运行时分析
  │                                              │
  └──────────────────────────────────────────────┘
              持续反馈和改进（管道是闭环）
```

**关键点：** 管道不是单向的，每个阶段都有分析和反馈 ^[extracted]。

## CICS Java开发支持（CICS TS V5.6+）

- 支持Maven和Gradle作为Java应用的构建框架
- CICS Bundle简化应用打包和部署
- JCICSX API允许Java开发者本地运行CICS应用（连接真实CICS Region测试）
- 应用可以在IDE中运行，测试完成后无需修改即可部署到真实CICS Region

## 相关概念

- [[concepts/cics-overview]] — CICS基础
- [[concepts/cics-java-modernization]] — CICS Java开发
- [[concepts/cics-async-event-processing]] — CICS异步编程和事件处理
- [[references/modernizing-applications-ibm-cics]] — 原始红皮书来源

## 来源

- IBM Redbooks REDP-5628-00, Chapter 8 "DevOps and IBM CICS", December 2020