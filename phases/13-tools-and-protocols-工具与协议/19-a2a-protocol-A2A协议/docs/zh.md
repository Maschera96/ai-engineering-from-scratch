# A2A — Agent-to-Agent 协议

> MCP 是智能体到工具（agent-to-tool）。A2A（Agent2Agent）是智能体到智能体（agent-to-agent）——一个让构建在不同框架上的不透明智能体协作的开放协议。由 Google 于 2025 年 4 月发布，2025 年 6 月捐赠给 Linux Foundation，并在 2026 年 4 月达到 v1.0，拥有 150+ 支持者，包括 AWS、Cisco、Microsoft、Salesforce、SAP 和 ServiceNow。它吸收了 IBM 的 ACP，并加入了 AP2 支付扩展。本课讲解 Agent Card、Task 生命周期，以及两种传输绑定。

**类型：** Build
**语言：** Python（stdlib，Agent Card + Task 框架）
**前置：** Phase 13 · 06（MCP 基础）、Phase 13 · 08（MCP 客户端）
**时间：** ~75 分钟

## 学习目标

- 区分智能体到工具（MCP）与智能体到智能体（A2A）的使用场景。
- 在 `/.well-known/agent.json` 发布带有技能和端点元数据的 Agent Card。
- 走通 Task 生命周期（submitted → working → input-required → completed / failed / canceled / rejected）。
- 使用带 Parts（text、file、data）的 Messages，以及作为输出的 Artifacts。

## 问题

一个客服智能体需要把写报告的工作委派给一个专门的写作智能体。A2A 之前的选项：

- 自定义 REST API。可行，但每一对配对都是一次性的。
- 共享代码库。要求两个智能体运行同一框架。
- MCP。不合适：MCP 用于调用工具，而不是让两个智能体在各自保留不透明内部推理的情况下协作。

A2A 填补了这一空白。它把交互建模为一个智能体向另一个智能体发送 Task，带有生命周期、消息和制品。被调用智能体的内部状态保持不透明——调用方只看到任务状态转换和最终输出。

A2A 是"让跨框架的智能体彼此对话"的协议。它不取代 MCP；两者是互补的。

## 概念

### Agent Card

每个符合 A2A 的智能体都在 `/.well-known/agent.json` 发布一张卡片：

```json
{
  "schemaVersion": "1.0",
  "name": "research-agent",
  "description": "Summarizes academic papers and drafts citations.",
  "url": "https://research.example.com/a2a",
  "version": "1.2.0",
  "skills": [
    {
      "id": "summarize_paper",
      "name": "Summarize a paper",
      "description": "Read a paper PDF and produce a 3-paragraph summary.",
      "inputModes": ["text", "file"],
      "outputModes": ["text", "artifact"]
    }
  ],
  "capabilities": {"streaming": true, "pushNotifications": true}
}
```

发现是基于 URL 的：获取卡片，得知 A2A 端点的 URL，枚举技能。

### 签名的 Agent Card（AP2）

AP2 扩展（2025 年 9 月）为 Agent Card 增加了加密签名。发布者用 JWT 签署自己的卡片；消费者验证。可防止冒充。

### Task 生命周期

```
submitted -> working -> completed | failed | canceled | rejected
             -> input_required -> working (loop via message)
```

客户端通过 `tasks/send` 发起。被调用智能体在各状态间转换；客户端通过 SSE 订阅状态更新或轮询。

### Messages 和 Parts

一条消息携带一个或多个 Parts：

- `text` — 纯文本内容。
- `file` — 带 mimeType 的 base64 blob。
- `data` — 带类型的 JSON 负载（给被调用智能体的结构化输入）。

示例：

```json
{
  "role": "user",
  "parts": [
    {"type": "text", "text": "Summarize this paper."},
    {"type": "file", "file": {"name": "paper.pdf", "mimeType": "application/pdf", "bytes": "..."}},
    {"type": "data", "data": {"targetLength": "3 paragraphs"}}
  ]
}
```

### Artifacts

输出是 Artifacts，而不是裸字符串。一个 Artifact 是一个命名的、带类型的输出：

```json
{
  "name": "summary",
  "parts": [{"type": "text", "text": "..."}],
  "mimeType": "text/markdown"
}
```

Artifacts 可以分块流式传输。调用方负责累积。

### 两种传输绑定

1. **JSON-RPC over HTTP。** `/a2a` 端点，POST 用于请求，可选 SSE 用于流式传输。默认绑定。
2. **gRPC。** 用于以 gRPC 为原生的企业环境。

两种绑定承载相同的逻辑消息形状。

### 不透明性保留

一个关键设计原则：被调用智能体的内部状态是不透明的。调用方看到任务状态和制品。被调用智能体的思维链、它的工具调用、它的子智能体委派——全都不可见。这与 MCP 不同，在 MCP 中工具调用是透明的。

理由：A2A 让竞争对手能够在不暴露内部实现的情况下协作。A2A 可以是"调用这个客服智能体"，而调用方无需了解该智能体如何实现该服务。

### 时间线

- **2025-04-09。** Google 宣布 A2A。
- **2025-06-23。** 捐赠给 Linux Foundation。
- **2025-08。** 吸收 IBM 的 ACP。
- **2025-09。** AP2 扩展（Agent Payments）发布。
- **2026-04。** v1.0 发布，拥有 150+ 支持组织。

### 与 MCP 的关系

| 维度 | MCP | A2A |
|-----------|-----|-----|
| 使用场景 | 智能体到工具 | 智能体到智能体 |
| 不透明性 | 透明的工具调用 | 不透明的内部推理 |
| 典型调用方 | 智能体运行时 | 另一个智能体 |
| 状态 | 工具调用结果 | 带生命周期的 Task |
| 授权 | OAuth 2.1（Phase 13 · 16） | JWT 签名的 Agent Card（AP2） |
| 传输 | Stdio / Streamable HTTP | JSON-RPC over HTTP / gRPC |

当你想调用一个特定工具时用 MCP。当你想把一整个任务委派给另一个智能体时用 A2A。许多生产系统两者都用：一个智能体用 MCP 作为其工具层，用 A2A 作为其协作层。

## 动手用

`code/main.py` 实现了一个最小的 A2A 框架：一个研究智能体发布它的卡片，一个写作智能体接收一个带有 Parts（包括一个 PDF 和一条文本指令）的 `tasks/send`，经历 working → input_required → working → completed 的转换，并返回一个文本制品。全部使用 stdlib；使用内存传输以聚焦于消息形状。

需要关注的地方：

- Agent Card JSON 形状。
- Task id 分配与状态转换。
- 带混合类型 Parts 的 Messages。
- 任务中途的 input-required 分支。
- 完成时的 Artifact 返回。

## 交付

本课产出 `outputs/skill-a2a-agent-spec.md`。给定一个应当可被其他智能体调用的新智能体，该技能产出 Agent Card JSON、技能模式和端点蓝图。

## 练习

1. 运行 `code/main.py`。追踪完整的 Task 生命周期，包括被调用智能体请求澄清的 input-required 暂停。

2. 添加一个签名的 Agent Card。用 HMAC 对卡片的规范化 JSON 进行签名。编写一个验证器，并确认它在卡片被篡改时会失败。

3. 实现任务流式传输：写作智能体通过 SSE 发出三个增量制品块，调用方累积它们。

4. 设计一个封装 MCP 服务器的 A2A 智能体。把每个 MCP 工具映射到一个 A2A 技能。注意权衡——失去了哪些不透明性？

5. 阅读 A2A v1.0 公告，找出截至 2026 年 4 月仍未被任何框架实现的那一个特性。（提示：它与多跳任务委派有关。）

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| A2A | "Agent-to-Agent 协议" | 用于不透明智能体协作的开放协议 |
| Agent Card | "`.well-known/agent.json`" | 描述智能体技能和端点的已发布元数据 |
| Skill | "一个可调用单元" | 智能体支持的命名操作（类比 MCP 工具） |
| Task | "委派单元" | 带生命周期和最终制品的工作项 |
| Message | "Task 输入" | 携带 Parts（text、file、data） |
| Part | "带类型的块" | 消息的 `text` / `file` / `data` 元素 |
| Artifact | "Task 输出" | 完成时返回的命名、带类型的输出 |
| AP2 | "Agent Payments Protocol" | 用于信任和支付的签名 Agent Card 扩展 |
| Opacity | "黑盒协作" | 被调用智能体的内部对调用方隐藏 |
| Input-required | "任务暂停" | 智能体需要更多信息时的生命周期状态 |

## 延伸阅读

- [a2a-protocol.org](https://a2a-protocol.org/latest/) — A2A 规范的权威来源
- [a2aproject/A2A — GitHub](https://github.com/a2aproject/A2A) — 参考实现与 SDK
- [Linux Foundation — A2A launch press release](https://www.linuxfoundation.org/press/linux-foundation-launches-the-agent2agent-protocol-project-to-enable-secure-intelligent-communication-between-ai-agents) — 2025 年 6 月治理移交
- [Google Cloud — A2A protocol upgrade](https://cloud.google.com/blog/products/ai-machine-learning/agent2agent-protocol-is-getting-an-upgrade) — 路线图与合作伙伴势头
- [Google Dev — A2A 1.0 milestone](https://discuss.google.dev/t/the-a2a-1-0-milestone-ensuring-and-testing-backward-compatibility/352258) — v1.0 发布说明与向后兼容指南
