# 综合实战 —— 构建完整的工具生态系统

> Phase 13 讲解了每一个组成部分。本综合实战将它们串联成一个具备生产形态的系统：一个暴露工具 + 资源 + 提示 + 任务 + UI 的 MCP 服务器、边缘处的 OAuth 2.1、一个 RBAC 网关、一个多服务器客户端、一次 A2A 子智能体调用、把追踪数据导入采集器的 OTel、CI 中的工具投毒检测，以及一个 AGENTS.md + SKILL.md 包。完成后，你能够为每一个架构选择做出辩护。

**类型：** 构建
**语言：** Python（标准库，端到端生态系统框架）
**前置要求：** Phase 13 · 01 到 21
**时长：** 约 120 分钟

## 学习目标

- 组合一个暴露工具、资源、提示和一个带 `ui://` 应用的任务的 MCP 服务器。
- 在服务器前端部署一个执行 RBAC 和固定哈希的 OAuth 2.1 网关。
- 编写一个使用 OTel GenAI 属性进行端到端追踪的多服务器客户端。
- 将部分工作负载委派给一个 A2A 子智能体；验证不透明性得到保留。
- 用 AGENTS.md + SKILL.md 打包整个技术栈，以便其他智能体能够驱动它。

## 问题

交付这套"研究并报告"系统：

- 用户提问："总结 2026 年 arXiv 上关于智能体协议、被引用最多的三篇论文。"
- 系统：通过 MCP 搜索 arXiv；通过 A2A 将论文摘要委派给一个专门的写作智能体；聚合结果；将一份交互式报告渲染为一个 MCP Apps `ui://` 资源；将每一步记录到 OTel。

Phase 13 中的所有原语都会登场。这不是一个玩具——2026 年由 Anthropic（Claude Research 产品）、OpenAI（带 Apps SDK 的 GPTs）以及第三方交付的生产级研究助手系统，正是这种形态。

## 概念

### 架构

```
[user] -> [client] -> [gateway (OAuth 2.1 + RBAC)] -> [research MCP server]
                                                      |
                                                      +- MCP tool: arxiv_search (pure)
                                                      +- MCP resource: notes://recent
                                                      +- MCP prompt: /research_topic
                                                      +- MCP task: generate_report (long)
                                                      +- MCP Apps UI: ui://report/current
                                                      +- A2A call: writer-agent (tasks/send)
                                                      |
                                                      +- OTel GenAI spans
```

### 追踪层级

```
agent.invoke_agent
 ├── llm.chat (kick off)
 ├── mcp.call -> tools/call arxiv_search
 ├── mcp.call -> resources/read notes://recent
 ├── mcp.call -> prompts/get research_topic
 ├── a2a.tasks/send -> writer-agent
 │    └── task transitions (opaque internals)
 ├── mcp.call -> tools/call generate_report (task-augmented)
 │    └── tasks/status polling
 │    └── tasks/result (completed, returns ui:// resource)
 └── llm.chat (final synthesis)
```

一个 trace id。每个 span 都带有正确的 `gen_ai.*` 属性。

### 安全态势

- OAuth 2.1 + PKCE，并使用资源指示符将受众固定到网关。
- 网关持有上游凭据；用户永远看不到它们。
- RBAC：`alice` 拥有 `research:read`、`research:write`，可以调用所有工具。`bob` 拥有 `research:read`，无法调用 `generate_report`。
- 固定描述清单：丢弃任何工具哈希发生变化的服务器。
- 二之规则（Rule of Two）审计：没有任何工具同时组合了不可信输入、敏感数据和有后果的操作。

### 渲染

最终的 `generate_report` 任务返回内容块外加一个 `ui://report/current` 资源。客户端的宿主（Claude Desktop 等）在沙盒化的 iframe 中渲染这个交互式仪表盘。仪表盘包含一个排序后的论文列表、引用计数，以及一个按钮——用户点击任意论文时，它会调用 `host.callTool('summarize_paper', {arxiv_id})`。

### 打包

整套东西按如下方式交付：

```
research-system/
  AGENTS.md                     # project conventions
  skills/
    run-research/
      SKILL.md                  # the top-level workflow
  servers/
    research-mcp/               # the MCP server
      pyproject.toml
      src/
  agents/
    writer/                     # the A2A agent
  gateway/
    config.yaml                 # RBAC + pinned manifest
```

用户通过 `docker compose up` 部署。Claude Code、Cursor、Codex 和 opencode 用户可以通过调用 `run-research` 技能来驱动这个系统。

### Phase 13 每节课的贡献

| 课程 | 综合实战所用内容 |
|--------|------------------------|
| 01-05 | 工具接口、提供商可移植性、并行调用、模式、linting |
| 06-10 | MCP 原语、服务器、客户端、传输、资源 + 提示 |
| 11-14 | 采样、roots + 征询（elicitation）、异步任务、`ui://` 应用 |
| 15-17 | 工具投毒、OAuth 2.1、网关 + 注册表 |
| 18 | A2A 子智能体委派 |
| 19 | OTel GenAI 追踪 |
| 20 | LLM 层的路由网关 |
| 21 | SKILL.md + AGENTS.md 打包 |

## 动手用

`code/main.py` 将前几节课的模式缝合成一个可运行的演示。全部基于标准库、全部在进程内，因此你可以从头到尾通读它。它运行了研究并报告场景的完整流程：与网关握手、模拟 OAuth 2.1、合并 tools/list、把 generate_report 作为任务、对 writer 的 A2A 调用、返回 ui:// 资源、发出 OTel span。

需要关注的内容：

- 跨越每一跳的一个 trace id。
- 网关策略阻止第二个用户写入。
- 任务生命周期从 working → completed，并同时返回文本和 ui:// 内容。
- A2A 调用的内部状态对编排者不透明。
- AGENTS.md 和 SKILL.md 是另一个智能体复现该工作流所需的唯一文件。

## 交付

本课产出 `outputs/skill-ecosystem-blueprint.md`。给定一个产品需求（研究、摘要、自动化），该技能产出完整架构：用哪些 MCP 原语、哪些网关控制、哪些 A2A 调用、哪些遥测、哪种打包方式。

## 练习

1. 运行 `code/main.py`。留意单一的 trace id 以及 span 如何嵌套。数一数演示触及了 Phase 13 中的多少个原语。

2. 扩展该演示：添加第二个后端 MCP 服务器（例如 `bibliography`），并确认网关将其工具合并到同一个命名空间中。

3. 用一个运行在子进程上的真实 A2A 写作智能体替换假的那个。使用第 19 课的框架。

4. 在编排者和 LLM 之间的路由网关中加入一个 PII 脱敏步骤。确认用户查询中的电子邮件被清除。

5. 为将要维护这个系统的队友编写一份 AGENTS.md。它应当在五分钟内读完，并为他们提供在 Cursor 或 Codex 中驱动综合实战所需的一切。

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| Capstone（综合实战） | "Phase-13 集成演示" | 使用每一个原语的端到端系统 |
| Research and report（研究并报告） | "这个场景" | 搜索、摘要、渲染模式 |
| Ecosystem（生态系统） | "所有部件合在一起" | 服务器 + 客户端 + 网关 + 子智能体 + 遥测 + 包 |
| Trace hierarchy（追踪层级） | "单一 trace id" | 每一跳的 span 共享该 trace；通过 span id 形成父子关系 |
| Gateway-issued token（网关签发的令牌） | "传递式认证" | 客户端只看到网关的令牌；网关持有上游凭据 |
| Merged namespace（合并的命名空间） | "所有工具在一个扁平列表里" | 在网关处的多服务器合并，冲突时加前缀 |
| Opacity boundary（不透明边界） | "A2A 调用隐藏内部细节" | 子智能体的推理对编排者不可见 |
| Three-layer stack（三层栈） | "AGENTS.md + SKILL.md + MCP" | 项目上下文 + 工作流 + 工具 |
| Defense-in-depth（纵深防御） | "多个安全层" | 固定哈希、OAuth、RBAC、二之规则、审计日志 |
| Spec compliance matrix（规范合规矩阵） | "我们交付的东西满足了规范的哪些要求" | 将交付物映射到 2025-11-25 要求的检查清单 |

## 延伸阅读

- [MCP — Specification 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25) —— 整合后的参考
- [MCP blog — 2026 roadmap](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/) —— 协议的发展方向
- [a2a-protocol.org](https://a2a-protocol.org/latest/) —— A2A v1.0 参考
- [OpenTelemetry — GenAI semconv](https://opentelemetry.io/docs/specs/semconv/gen-ai/) —— 权威的追踪约定
- [Anthropic — Claude Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview) —— 生产级智能体运行时模式
