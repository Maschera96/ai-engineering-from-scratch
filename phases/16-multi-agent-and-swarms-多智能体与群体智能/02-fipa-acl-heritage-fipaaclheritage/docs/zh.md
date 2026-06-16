# FIPA-ACL 和语音行为的传承

> 在MCP之前，在A2A之前，有FIPA-ACL。 2000 年，IEEE 智能物理代理基金会批准了一种具有 20 种执行语句、两种内容语言和一组交互协议的代理通信语言——合约网、subscribe/notify、request-when。它从行业中消失了，因为本体开销对于网络来说太重了，但多代理系统的法学硕士复兴正在悄悄地重新实现相同的想法，而无需正式的语义：JSON 合约代表执行性，自然语言代表本体。本课认真阅读 FIPA-ACL，这样您就可以看到哪些 2026 年协议决策是重新发明的、哪些是新颖的，以及当前浪潮将在哪些方面重新发现 2000 年代已经解决的问题。

**类型：** 学习
**语言：** Python (stdlib)
**先决条件：** 第 16 阶段·01（为什么使用多代理）
**时间：** ~60 分钟

＃＃ 问题

2026 年代理协议领域将非常繁忙：用于工具的 MCP、用于代理的 A2A、用于企业审计的 ACP、用于去中心化信任的 ANP、用于自然语言内容的 NLIP，以及 CA-MCP 和两打研究提案。每个规范都宣称自己是基础规范。

诚实的解读是，他们中的大多数人正在重新发现一个非常具体的二十年历史的决策树。 Austin (1962) 和 Searle (1969) 的言语行为理论告诉我们“言语就是行动”。 KQML (1993) 将其转变为有线协议。 FIPA-ACL（2000 年批准）制定了参考标准化：20 种执行语句、内容语言 SL0/SL1、合约网和订阅通知的交互协议。 JADE 和 JACK 是 Java 参考平台。 2010 年左右，这项努力逐渐消退，因为本体开销太大，而 Web 取得了胜利。

当您查看 MCP 的 `tools/call`、A2A 的任务生命周期或 CA-MCP 的共享上下文存储时，您看到的是 FIPA 决策的更软的、JSON 原生的重新哈希。了解传统可以告诉您两件事：哪些新的“创新”实际上是重新发明，以及新规范将重新发现哪些旧的故障模式。

＃＃ 概念

### 言语行为，在一个段落中

奥斯汀注意到有些句子并不能描述世界——它们改变了世界。 “我保证。” “我请求。” “我声明。”他称这些为表演性话语。塞尔将五类形式化：断言性、指示性、承诺性、表达性、声明性。 KQML（Finin 等人，1993）使软件代理可以实现这一点：消息是执行（动作）加上内容（动作是关于什么的）。 FIPA-ACL 清理了 KQML 的空白并标准化了大约 20 个执行语句。

### 20 项 FIPA 表演行为（部分列表）

| 述行性 | 意图 |
|---|---|
| XCodeToken0X | “我告诉你P是真的” |
| XCodeToken0X | “我要求你做X” |
| XCodeToken0X | “P是真的吗？” |
| XCodeToken0X | “X 的值是多少？” |
| XCodeToken0X | “我建议我们做X” |
| XCodeToken0X | “我接受这个提议” |
| XCodeToken0X | “我拒绝这个提议” |
| XCodeToken0X | “我同意做X” |
| XCodeToken0X | “我拒绝做X” |
| XCodeToken0X | “我确认P是真的” |
| XCodeToken0X | “我否认P” |
| XCodeToken0X | “您的消息未解析” |
| XCodeToken0X | “征集有关 X 的提案” |
| XCodeToken0X | “当 X 发生变化时通知我” |
| XCodeToken0X | “取消正在进行的X” |
| XCodeToken0X | “我尝试了 X 但失败了” |

完整列表位于 `fipa00037.pdf`（FIPA ACL 消息结构）中。重点不是要记住它——重点是它们中的每一个都对应于 LLM 协议最终重新添加的一个原语。

### 规范 FIPA-ACL 消息

```
(inform
  :sender       agent1@platform
  :receiver     agent2@platform
  :content      "((price IBM 83))"
  :language     SL0
  :ontology     finance
  :protocol     fipa-request
  :conversation-id   conv-42
  :reply-with   msg-17
)
```

七个字段携带协议信封；一个字段 (`content`) 携带有效负载。其余字段正是您每次将重试、线程和本体连接到 JSON 协议时重新发明的字段。

### 两个旧平台

**JADE**（Java 代理开发框架，1999-2020 年代）是最常用的符合 FIPA 的运行时。代理扩展了基类，交换 ACL 消息，在容器内运行，并使用“行为”进行协调。交互协议库附带有contract-net、subscribe-notify、request-when 和propose-accept。

**JACK**（面向代理的软件，商业）强调基于 FIPA 消息的 BDI（信念-愿望-意图）推理。更正式，更少采用。

一旦网络堆栈吞噬了多代理用例，两者都会下降。 MCP 和 A2A 是 2026 年的运行时“容器”。

### FIPA 为何衰落

- **本体开销。** FIPA 需要共享本体来解析 `content`。就本体论达成一致是一个长达数年的标准过程。 Web 只使用 HTTP + JSON。
- **没有人使用形式语义。** SL（语义语言）给出了严格的真值条件，但大多数生产系统使用自由形式的内容并忽略了形式主义。
- **工具锁定。** JADE 仅适用于 Java； JACK是商业的。通晓多种语言的团队围绕这两者展开讨论。
- **互联网赢得了堆栈。** REST，然后是 JSON-RPC，然后 gRPC 取代了 ACL 的传输。

### LLM 复兴是 FIPA-lite

将 FIPA `request` 与 MCP `tools/call` 进行比较：

```
(request                                {
  :sender  agent1                         "jsonrpc": "2.0",
  :receiver tool-server                   "method":  "tools/call",
  :content "(lookup stock IBM)"           "params":  {"name":"lookup_stock",
  :ontology finance                                   "arguments":{"symbol":"IBM"}},
  :conversation-id c42                    "id": 42
)                                        }
```

相同的信封，不同的语法。两者都携带：who、who、intent、payload、correlation id。两者都不是对对方的革命——它们是同一设计的不同权衡。

Liu 等人的 2025 年调查("A Survey of Agent Interoperability Protocols: MCP, ACP, A2A, ANP", arXiv:2505.02279) makes this lineage explicit: MCP corresponds to tool-use speech acts, A2A to agent-peer speech acts, ACP to audit-trail speech acts, ANP to decentralized-identity extensions.新规范是具有 JSON 语法和更宽松语义的 ACL 后代。

### 明确说明的权衡

**FIPA 为您提供了什么以及现代规格下降：**

- 形式语义 - 您可以证明 `inform` 意味着发送者相信该内容。
- 规范的表演目录 - 你不必重新争论“我们应该有一个 `cancel` 吗？”。
- 数十年的交互协议模式——合同网络、订阅通知、提议接受——具有已知的正确性属性。

**现代规范为您提供了哪些 FIPA 没有提供的功能：**

- JSON 原生有效负载与所有现代工具兼容。
- 法学硕士无需手动编码本体即可解释的自然语言内容。
- Web 堆栈传输（HTTP、SSE、WebSocket）。
- 通过自描述文档（MCP `listTools`、A2A 代理卡）发现能力。

更宽松的意图语义，更容易实现。这就是确切的交易。

### 值得移植的交互协议

FIPA 发布了大约 15 种交互协议。其中三个值得在LLM多智能体系统中发扬光大：

1. **合约网络协议（CNP）。** 管理者发布`cfp`（征集提案）；投标人用 `propose` 响应； manager accepts/rejects. 这是典型的任务市场模式（第 16 阶段·16 协商）。
2. **Subscribe/Notify.** 订阅者发送`subscribe`；每当主题发生变化时，发布者都会发送 `inform`。这是 2026 年的每一个事件总线。
3. **Request-When.** “当条件 Y 成立时执行 X。”有先决条件的延迟行动。 2026 年的模拟是持久工作流引擎中的延迟任务（第 16·22 阶段生产扩展）。

每个都清晰地映射到现代消息队列、HTTP + 轮询或 SSE 流。

### 当你放弃本体时会发生什么

如果没有共享本体，智能体就可以从自然语言内容中推断出含义。记录的 2026 故障模式是 **语义漂移**：两个代理使用相同的单词 (`"customer"`) 来表示略有不同的概念，接收者的代理按照错误的解释进行操作，没有模式验证器捕获它。 FIPA 的本体要求将在解析时拒绝该消息。

无需完整本体的缓解措施：

- `content` 上的 JSON 模式 — 拒绝线路上的结构错误。
- 键入的工件 (A2A) — 拒绝错误的模式。
- 信封中明确的表演性——即使内容是自然语言，也使意图明确。

### 2026 年规范，映射到言语行为遗产

| 现代规格 | FIPA 模拟 | 它保留了什么 | 它掉落什么 |
|---|---|---|---|
| MCP `tools/call` | XCodeToken0X | 显式意图，相关 ID | 形式语义、本体论 |
| MCP `resources/read` | XCodeToken0X | 显式意图，相关 ID | 形式语义 |
| A2A 任务生命周期 | 合约网+请求时 | 异步生命周期、状态转换 | 正式的完整性保证 |
| A2A 流事件 | XPATHToken0X | 异步推送 | 类型谓词订阅 |
| CA-MCP 共享上下文 | 黑板（Hayes-Roth 1985） | 多写入者共享内存 | 逻辑一致性模型 |
| 自然语言处理 | 自然语言内容 | LLM本地人 | 图式 |

从上到下阅读表格，模式是：保留结构原语，放弃形式主义，让法学硕士掩盖歧义。

## 构建它

`code/main.py` 实现纯标准库 FIPA-ACL 转换器。它对规范 ACL 信封进行编码和解码，并显示每个 MCP / A2A 消息形状如何简化为相同的七个字段。演示：

- 将五个 MCP 样式和 A2A 样式消息编码为 FIPA-ACL。
- 将 FIPA-ACL 解码回现代等效内容。
- 使用 `cfp`、`propose`、`accept-proposal`、`reject-proposal` 在一名经理和三名投标人之间运行玩具合同网络谈判。

跑步：

```
python3 code/main.py
```

输出是并排跟踪，显示每条现代消息的 2026 JSON 形式和 FIPA-ACL 形式，然后是合约净出价的往返。相同的协议原语在往返过程中仍然存在；只是语法不同。

## 使用它

`outputs/skill-fipa-mapper.md` 是一种读取任何代理协议规范并生成 FIPA-ACL 映射的技能。在采用新协议之前使用它来回答：“这是真正的新协议，还是带有 JSON 语法的 `inform`？”

## 发货

不要带回 FIPA-ACL。带回其清单：

- 每条消息的意图原语（执行）是什么？
- 请求-响应和取消是否有关联 ID？
- 是否有明确的内容语言（JSON-RPC、纯文本、结构化类型工件）？
- 交互协议是一流的，还是您从头开始重新实现合约网？
- 当两个代理对内容含义（语义漂移）存在分歧时会发生什么？

在将任何新协议投入生产之前，记录这五个问题。

## 练习

1.运行`code/main.py`。观察往返编码。确定哪个 FIPA 执行对应于 `tools/call`、`resources/read` 和 A2A 任务创建。
2. 使用 `cancel` 执行式扩展合约网络演示，让管理者在中标时撤回任务。 `cancel` 可以解决哪些单独重试无法解决的故障情况？
3. 阅读 FIPA ACL 消息结构 (http://www.fipa.org/specs/fipa00037/) 第 4.1–4.3 节。选择本课程中未涵盖的一个执行行为并描述其现代 JSON-RPC 类似物。
4. 阅读 Liu 等人，arXiv:2505.02279。对于每个 MCP、A2A、ACP、ANP，列出它们保留和删除的 FIPA 执行系列。
5. 在您自己的系统中为 `request` 执行的 `content` 字段设计一个最小的 JSON 架构。该模式为您提供了哪些纯自然语言无法提供的功能，以及它的成本是多少？

## 关键术语

| 学期 | 人们怎么说 | 它实际上意味着什么 |
|------|----------------|------------------------|
| 言语行为 | “一句话能做某事” | Austin/Searle：言语即行动。 ACL 的理论父辈。 |
| FIPA | “那个古老的 XML 东西” | IEEE 智能物理代理基金会。 2000年标准化ACL。 |
| 前交叉韧带 | 《代理沟通语言》 | FIPA 的信封格式：表演+内容+元数据。 |
| 述行性 | “动词” | 消息的意图类：`inform`、`request`、`propose`、`cfp` 等。 |
| 韩国QML | “FIPA 的前身” | 知识查询和操作语言（1993）。更简单，更窄。 |
| 本体论 | 《共享词汇》 | 内容语言所讨论的概念的正式定义。 |
| SL0/SL1 | “FIPA 内容语言” | 语义语言级别 0 和 1 — 形式内容语言族。 |
| 合约网 | 《任务市场》 | 经理签发cfp；投标人提出建议；经理接受。规范的交互协议。 |
| 交互协议 | “消息模式” | 具有已知正确性的一系列执行语句：请求时间、订阅通知等。 |

## 进一步阅读

- [刘等人。 — 代理互操作性协议调查：MCP、ACP、A2A、ANP](https://arxiv.org/html/2505.02279v1) — 将现代规范与 FIPA 传统联系起来的 2025 年规范调查
- [FIPA ACL 消息结构规范 (fipa00037)](http://www.fipa.org/specs/fipa00037/) — 批准的 2000 信封格式
- [FIPA 交流行为库规范 (fipa00037)](http://www.fipa.org/specs/fipa00037/) — 完整的表演目录
- [MCP 规范 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25) — XCODETOKEN1X/XCODETOKEN2X 的现代工具使用等效项
- [A2A 规范](https://a2a-protocol.org/latest/specification/) — 现代代理对等体的合约网和订阅通知
