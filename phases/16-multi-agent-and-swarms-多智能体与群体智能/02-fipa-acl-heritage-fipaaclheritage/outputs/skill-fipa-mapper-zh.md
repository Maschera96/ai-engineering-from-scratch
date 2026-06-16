---
name: fipa-mapper-zh
description: 将任何 2026 年代理协议规范（MCP、A2A、ACP、ANP、CA-MCP、NLIP 或新规范）映射到 FIPA-ACL 执行和交互协议，以确定什么是真正的新颖性和什么是重新发明。
version: 1.0.0
phase: 16
lesson: 02
tags: [multi-agent, protocols, FIPA, speech-acts, interoperability]
---

给定一个新的代理协议规范，生成 FIPA-ACL 映射，以便读者可以辨别哪些部分是重新发明的，哪些是真正的新结构。

生产：

1. **信封映射。** 对于规范定义的每种消息类型，命名最接近的 FIPA 执行式（`inform`、`request`、`query-if`、`query-ref`、`propose`、`accept-proposal`、`reject-proposal`、`cfp`、`subscribe`、`cancel`、`failure`、 `not-understood`，或其他 ~20 之一）。如果没有表现上的契合，请准确描述差距。
2. **关联模型。** 规范如何将请求与回复、取消与原始请求以及流式事件与订阅关联起来？与 FIPA 的 `:conversation-id` 和 `:reply-with` 字段进行比较。
3. **内容语言立场。** 规范是否强制要求内容模式（类型化工件、JSON 模式）、接受自然语言还是保持开放？与 FIPA 的 SL0/SL1 和本体字段进行比较。
4. **交互协议库。** 哪些 FIPA 交互协议可以在规范之上实现：contract-net、subscribe-notify、request-when、propose-accept？命名将实现每个消息的消息。
5. **发现模型。** 代理如何找到交易对手和能力（MCP `listTools`、A2A 代理卡、ANP DID + 元协议）？与 FIPA 的目录服务程序和黄页服务进行比较。
6. **重塑与新颖。** 生成一个包含三列的简短表格：[FIPA 概念、现代规范等效项、更改内容]。将每一行标记为[重新发明]或[新颖结构]。仅当规范引入 FIPA 所没有的原语时，行才是“新颖结构”——去中心化身份、类型化多模式工件和 LLM 可解释的内容是常见的候选者。

硬拒绝：

- 任何声称规范具有“革命性”但未显示原始 FIPA 所没有的映射。言语行为理论+本体开销是失败模式，而不是原语。
- 忽略发现层的框架比较。没有发现的规范是不完整的，不是新颖的。
- 诸如“Protocol X 取代 FIPA”之类的陈述没有解决当两个代理对内容含义存在分歧（语义漂移）时会发生什么情况。

拒绝规则：

- 如果规范是预标准化的（草案 < 6 个月前，没有公开实现），请声明映射是临时的并标记三个最有可能的更改。
- 如果规范是闭源的或仅限企业使用（某些 ACP 风格），请映射记录的内容并指出差距。
- 如果用户仅提供博客文章（没有规范文档），请在映射之前询问规范。

输出：一页的简介。从单句摘要开始（“协议 X 是具有 JSON 语法和基于 DID 的发现层的 FIPA XCODETOKEN0X/XCODETOKEN1X。”），然后是上面的六个部分，然后是结束段落回答：“此规范将重新发现哪种旧的 FIPA 故障模式？”
