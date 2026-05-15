---
title: Modernizing Applications with IBM CICS
category: references
tags: [cics, ibm, zos, mainframe, redbook]
sources: [/home/openclaw/下载/CICS.pdf]
summary: IBM红皮书REDP-5628-00（2020年12月），覆盖CICS应用现代化方法：API构建、Java重写、Channels/Containers、DevOps流水线。
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

# Modernizing Applications with IBM CICS

## 概述

IBM Redbooks REDP-5628-00（2020年12月），作者：Russell Bonner、Sophie Green、Ezriel Gross、Jim Harrison、Debra Scharfstein、Will Yates。本书聚焦于CICS应用的现代化，使现有CICS应用能够与云原生应用集成。

## 核心主题

本书采用一个传统的工资单（Payroll）应用作为示例，逐步展示以下现代化技术：

1. **Channels and Containers** — 将传统COMMAREA迁移到更灵活的Channels/Containers接口
2. **Java现代化** — 在CICS中引入Java组件，使用JCICSX API调用现有COBOL程序
3. **API暴露** — 将CICS业务逻辑暴露为RESTful服务
4. **DevOps流水线** — 在IBM Z上建立CI/CD流水线

## 三种现代化路径

| 路径 | 描述 | 风险/成本 |
|---|---|---|
| 维持现状 | 保持现有应用不变 | 应用逐渐落后于技术发展 |
| 云原生迁移 | 将应用完全迁移到云端 | 高风险、高成本，可能丢失CICS的交易能力 |
| **现代化现有应用** | 构建API、重写部分组件、使用CICS新能力 | 中等风险，保留核心业务逻辑 |

## 配套课程

本书与在线课程"Modernize applications with IBM CICS"配套提供，包含视频讲座和动手实验。

## 相关概念

- [[concepts/cics-overview]] — CICS概述与50年历史
- [[concepts/cics-application-development]] — CICS应用开发范式
- [[concepts/cics-java-modernization]] — Java在CICS中的现代化应用
- [[concepts/cics-devops]] — IBM Z上的DevOps实践

## 来源

- IBM Redbooks REDP-5628-00, *Modernizing Applications with IBM CICS*, December 2020