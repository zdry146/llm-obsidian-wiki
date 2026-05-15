---
title: WebSphere MQ Programming
category: concepts
tags: [ibm, websphere, programming, mqi, mqput, mqget, cobol, java, triggering]
sources: [/home/openclaw/下载/ibm mq.pdf]
summary: WebSphere MQ应用程序通过MQI（Message Queue Interface）编写，主要动词包括MQCONN/MQDISC连接管理、MQOPEN/MQCLOSE对象管理、MQPUT/MQGET消息操作，以及事务控制（MQCMIT/MQBACK）。支持C/COBOL/Java/C#等语言。
provenance:
  extracted: 0.82
  inferred: 0.13
  ambiguous: 0.05
base_confidence: 0.67
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15
updated: 2026-05-15
---

# WebSphere MQ Programming

## MQI概览

MQI（Message Queue Interface）目前有26个动词 ^[extracted]。WebSphere MQ V7.0之前约13个，V7.0引入发布/订阅和消息属性操作后增加了一倍 ^[extracted]。

主要动词分类：

### 连接管理
- **MQCONN / MQCONNX**：连接到队列管理器 ^[extracted]。MQCONNX支持更多连接控制参数（如安全套接字配置）。连接是MQI中最耗时的操作，应尽量减少使用次数 ^[extracted]
- **MQDISC**：断开连接。若有未提交事务，自动提交 ^[extracted]

### 对象管理
- **MQOPEN**：打开队列或主题，使其可被访问。参数包括对象名和使用方式标志（如MQOO_OUTPUT/MQOO_INPUT_AS_Q_DEF） ^[extracted]
- **MQCLOSE**：关闭对象 ^[extracted]

### 消息操作
- **MQPUT**：发送消息到队列或发布到主题 ^[extracted]
- **MQGET**：从队列接收消息或接收订阅出版物。可设置等待超时（MQGMO_WAIT） ^[extracted]
- **MQPUT1**：MQOPEN + MQPUT + MQCLOSE的简写形式，适合一次性向唯一队列发消息的高效操作 ^[extracted]
- **MQSUB**：创建对主题的订阅 ^[extracted]

### 对象查询/设置
- **MQINQ**：查询对象属性 ^[extracted]
- **MQSET**：设置对象属性 ^[extracted]

### 事务控制
- **MQBEGIN**：开始全局协调（XA）工作单元 ^[extracted]
- **MQCMIT**：提交同步点，所有工作单元消息操作生效 ^[extracted]
- **MQBACK**：回滚所有操作，消息恢复原状态 ^[extracted]

## MQI返回码

每个MQI调用都返回两个参数 ^[extracted]：
- **CompCode**：简单成功/警告/失败指示
- **Reason**：详细原因码

典型Reason码示例：`2033（MQRC_NO_MSG_AVAILABLE）` — 队列中没有消息 ^[extracted]。

## 典型代码模式

以下代码片段展示经典请求/响应模式 ^[extracted]（对应原书图3-5）：

```
1. MQCONN(QMName, &HCon, &CompCode, &Reason)       // 连接队列管理器
2. MQOPEN(QUEUE1, MQOO_OUTPUT, &HObj1, ...)       // 打开发送队列
3. MQOPEN(QUEUE2, MQOO_INPUT_AS_Q_DEF, &HObj2, ...)// 打开接收队列
4. MQPUT(HObj1, &md, &pmo, buffer, ...)           // 发送请求
5. MQCLOSE(HObj1, ...)                            // 关闭发送队列
6. MQGET(HObj2, &md, &gmo, ...)                   // 等待响应（最多10秒）
7. MQCLOSE(HObj2, ...)                            // 关闭接收队列
8. MQDISC(&HCon, ...)                             // 断开连接
```

关键点：
- 发送请求时将回复队列名填入`md.ReplyToQ`
- `MQPMO_NO_SYNCPOINT`标志显式指定不启动事务
- `MQGMO_WAIT`使MQGET最多等待10秒，无消息时返回2033
- 使用MsgID/CorrelID匹配请求和响应 ^[extracted]

## 消息属性和JMS支持

JMS（Java Message Service）是Java EE标准的消息API ^[extracted]。通过WebSphere MQ：
- JEE应用服务器可以选择任意厂商的JMS实现
- WebSphere Application Server内置MQ客户端运行时代码和管理面板，便于连接MQ队列管理器 ^[extracted]

## 触发（Triggering）

触发机制使应用程序仅在有工作时才启动 ^[extracted]，避免应用程序长期运行等待消息。

触发工作流：
1. 程序A向应用队列A_Q发送消息
2. 队列管理器发现A_Q已触发，将触发消息写入**初始化队列**
3. **触发监视器**从初始化队列获取触发消息
4. 触发监视器启动程序B
5. 程序B从A_Q获取实际消息进行处理 ^[extracted]

触发类型 ^[extracted]：
- **FIRST**：队列从空变为非空时触发（推荐）
- **EVERY**：每条消息都触发
- **n条消息**：队列达到n条消息时才触发

被触发应用的设计规则：必须在短超时内循环读取队列，直到队列为空 ^[extracted]。

## z/OS特定集成

在z/OS上，MQ提供与CICS和IMS的**桥接** ^[extracted]：
- 将MQ消息转换为CICS事务的输入
- 转换CICS事务的响应为MQ消息
- 既有应用无需修改即可接入MQ基础设施 ^[extracted]

在z/OS上，全局事务由**CICS、IMS或RRS**协调，应用程序不应调用MQCMIT/MQBACK ^[extracted]。

## 相关概念

- [[concepts/websphere-mq-core-concepts]] — MQI和消息风格
- [[concepts/websphere-mq-messages]] — MQMD消息描述符
- [[concepts/websphere-mq-objects]] — 触发相关对象（初始化队列、进程对象）
- [[concepts/websphere-mq-configuration]] — 配置触发器和队列
- [[concepts/cics-overview]] — CICS与MQ的桥接机制
- [[concepts/zos-overview]] — z/OS上MQ的事务协调