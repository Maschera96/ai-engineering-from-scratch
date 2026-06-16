---
name: primitive-splitter-zh
description: 将 MCP 服务器草案中的每项能力归类为 tool、resource 或 prompt，并给出理由。
version: 1.0.0
phase: 13
lesson: 10
tags: [mcp, primitives, resources, prompts]
---

给定一个待构建的 MCP 服务器的能力清单（以自然语言描述或工具草案列表形式），将每一项归类为 tool、resource 或 prompt，并给出一句话理由。

产出：

1. 逐能力归类。对每一项，返回 `{name, primitive: tool | resource | prompt, rationale}`。
2. 资源 URI 方案。如果有能力被归为资源，提出一套 URI 方案（`notes://`、`gh://`、`db://`）和一个模板模式。
3. Prompt 参数骨架。如果有能力被归为 prompt，提出参数列表以及必选/可选标记。
4. 订阅候选项。标记那些频繁变化、能从 `resources/subscribe` 获益的资源。
5. 反模式标记。指出那些旧设计把读取操作包装成工具（例如 `notes_read(id)`）、而用资源会更合适的情况。

硬性拒绝：
- 任何被归类为“既是 tool 又是 resource”而未做拆分的能力。要么二选一，要么搭建一对。
- 任何未明确必选参数的 prompt。要在斜杠命令 UI 中呈现，需要参数模式（schema）。
- 任何不可寻址的资源 URI 方案（自由格式字符串，而非 URI）。

拒绝规则：
- 如果所有能力都落为 tool，则拒绝，并询问该服务器是否有只读数据可以作为资源。
- 如果没有能力适合作为 prompt，那也没关系；prompt 是可选的。不要凭空编造。
- 如果该服务器的领域更适合用 A2A（智能体间协作、不透明状态）来服务，则拒绝并重定向到 Phase 13 · 19。

输出：一页决策报告，包含归类表格、URI 方案提议、prompt 骨架和订阅标记。最后给出对该服务器最具影响力的那一个 tool -> resource 转换。
