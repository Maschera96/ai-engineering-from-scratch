---
name: mcp-handshake-tracer-zh
description: 给定一段 pcap 风格的 MCP 客户端-服务器对话记录，为每条消息标注其原语、生命周期阶段以及能力依赖。
version: 1.0.0
phase: 13
lesson: 06
tags: [mcp, json-rpc, lifecycle, capabilities]
---

给定一段从 MCP 会话中捕获的 JSON-RPC 2.0 信封序列，生成一份逐步讲解，为每条消息命名其原语、生命周期阶段以及底层能力标志。

生成内容：

1. 逐条消息标注。对每个 `{request, response, notification}`，说明：方向（客户端到服务器或服务器到客户端）、原语（tools / resources / prompts / roots / sampling / elicitation / lifecycle）、生命周期阶段，以及为使该消息有效而必须协商的能力标志。
2. 能力检查。从记录中重建 `initialize` 交换过程，并列出所有已协商的能力。标记出任何会违反某个缺失能力的消息。
3. 错误诊断。对每个 JSON-RPC 错误，根据上下文命名其代码以及最可能的原因。
4. 完整性审计。标记出缺失以下任一项的记录：`initialize`、`initialized` 通知、至少一个 `tools/list` 或等价项、优雅关闭。
5. 规范合规性。对照 2025-11-25 规范的最小字段集，检查每个请求的 params。标记出遗漏项。

硬性拒绝：
- 任何使用规范允许集之外的方法且无 `x-` 前缀的消息。
- 客户端未声明 `sampling` 能力时出现的任何 `sampling/createMessage` 消息。
- `notifications/initialized` 到达之前的任何调用。

拒绝规则：
- 如果被要求审计来自非 MCP 协议的记录，拒绝并指向 A2A 规范（Phase 13 · 19）作为替代方案。
- 如果被要求“修复”记录，拒绝。本技能负责标注；不负责重写。将更正交由实现该协议的 SDK 处理。

输出：按到达顺序，每条消息一行标注：`[phase/primitive/capability] <method or result shape>`。以三行摘要结尾，说明任何能力违规以及任何缺失的生命周期步骤。
