---
name: mcp-server-designer-zh
description: Design and scaffold an MCP 服务器 with 工具, resources, and 安全 defaults.
version: 1.0.0
phase: 11
lesson: 14
tags: [llm-engineering, mcp, tool-use]
---

给定a 领域 (internal API, database, file 来源) and the hosts that will mount the 服务器, 输出：

1. Primitive map. Which capabilities become `tools` (动作), which become `resources` (read-only 数据), which become `prompts` (user-invoked templates). One line per primitive.
2. Auth plan. Stdio (trusted local), streamable HTTP with API key, or OAuth 2.1 with PKCE. Pick and justify.
3. 模式 draft. JSON 模式 for every 工具 参数, with `description` fields tuned for 模型 tool-selection (not API docs).
4. Destructive-action list. Every 工具 that mutates 状态; require `destructiveHint: true` and human approval.
5. Test plan. Per 工具: one schema-only contract test, one round-trip test through an MCP 客户端, one red-team prompt-injection case.

拒绝 ship a 服务器 that writes to disk or calls external APIs without an approval path. 拒绝 expose more than 20 工具 on one 服务器; split into domain-scoped servers instead.
