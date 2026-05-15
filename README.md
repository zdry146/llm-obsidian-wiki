# llm-obsidian-wiki

> 基于 Obsidian + LLM 构建的 IBM 大型机（z/OS）知识库

## 仓库结构

```
├── concepts/      # 核心概念笔记（51篇）
├── references/    # 参考资料（官方文档/Redbooks 摘录）
├── skills/        # 技能文档
└── synthesis/     # 综合整理与架构设计
```

## 知识体系

覆盖 IBM 大型机完整技术栈：

| 领域 | 内容 |
|------|------|
| **z/OS** | 操作系统、硬件架构、Parallel Sysplex、工作负载管理 |
| **JCL** | 作业控制语言、数据集管理、条件执行、调度与检查点重启 |
| **CICS** | 事务处理、COBOL 编程、EXEC API、Channels/Containers、Java 现代化 |
| **DB2 12 for z/OS** | 数据库管理、性能优化、高可用、SQL 增强、时间表 |
| **IBM MQ / WebSphere MQ** | 消息中间件、MQI 编程、发布/订阅、队列管理、与 CICS/z/OS 集成 |

## 资料来源

- IBM Redbooks（SG24、REDP 系列）
- IBM z/OS MVS JCL User's Guide (SA23-1386)
- IBM DB2 12 for z/OS Technical Overview

## 使用说明

- 用 **Obsidian** 打开本仓库即可浏览
- 通过 **LLM** 检索和整理知识体系
- 持续更新中，欢迎补充

## 最近更新

- 2026-05: 完善 MQ/MOM 知识体系、CICS 应用现代化、z/OS Batch 处理
