---
name: mcp-server-scaffolder-zh
description: 为特定领域脚手架搭建一个 MCP server，合理划分 tools/resources/prompts，并提供迁移到 SDK 的进阶路径。
version: 1.0.0
phase: 13
lesson: 07
tags: [mcp, server, fastmcp, scaffold]
---

给定一个领域（笔记、工单、文件、数据库，等等），产出一份 MCP server 规划：哪些能力作为 tools 暴露，哪些作为 resources，哪些作为 prompts，再加上迁移到 Python 或 TypeScript SDK 的进阶路径。

产出内容：

1. Tools 清单。用户明确要求执行的原子操作。包含名称、描述（Use-when 模式）、输入 schema 以及注解提示。
2. Resources 清单。用户想要读取的数据。URI 方案、mime 类型，以及是否启用 `resources/subscribe`。
3. Prompts 清单。host 应作为斜杠命令暴露的可复用模板。参数列表。
4. 能力声明。server 在 `initialize` 中返回的确切 `capabilities` 对象。
5. 进阶说明。每一部分对应的 FastMCP（Python）或 TypeScript SDK 等价实现。指出其中一个能替代脚手架里手写 stdlib 模式的 SDK 特性（例如 `lifespan`、`context`）。

硬性拒绝：
- 任何只作为 tool、却没有作为 resource 暴露的「数据库查询」。正确的划分是：`/list` 和 `/read` 用 resource，带参数的 `/query` 用 tool。
- 任何在同一命名空间中把用户输入类 tool 与特权类 tool 混在一起、且没有注解的 server。
- 任何在没有持久化通知机制的情况下就声称具备 `resources/subscribe` 能力的 server 脚手架。

拒绝规则：
- 如果该领域没有只读面，拒绝脚手架搭建 resources；建议改为仅含 tool 的 server。
- 如果该领域没有天然的斜杠命令模板，拒绝脚手架搭建 prompts。
- 如果用户要求某种鉴权方案，拒绝并引导至 Phase 13 · 16（OAuth 2.1）。

输出：一页纸的 server 规划，包含三类基元清单、capability 对象，以及一段 10 行的 `@app.tool()` 装饰器风格的进阶示例片段。最后以 server 应当设置的那个最重要的单个注解标志收尾。
