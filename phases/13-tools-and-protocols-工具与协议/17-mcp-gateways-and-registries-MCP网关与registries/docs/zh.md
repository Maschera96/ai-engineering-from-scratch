# MCP 网关与 Registries — 企业控制平面

> 企业不能任由每个开发者随意安装 MCP 服务器。网关集中处理认证、RBAC、审计、限流、缓存和工具投毒检测，然后将合并后的工具面以单一 MCP 端点对外暴露。Official MCP Registry（由 Anthropic + GitHub + PulseMCP + Microsoft 维护，经过命名空间验证）是规范的上游来源。本课说明网关适用于何处，演示一个最小实现，并概览 2026 年的厂商格局。

**类型：** Learn
**语言：** Python（标准库，最小网关）
**前置：** Phase 13 · 15（工具投毒）、Phase 13 · 16（OAuth 2.1）
**时长：** 约 45 分钟

## 学习目标

- 解释 MCP 网关所处的位置（位于 MCP 客户端与多个后端 MCP 服务器之间）。
- 实现网关的五项职责：认证、RBAC、审计、限流、策略。
- 在网关层强制执行一份固定工具哈希（pinned-tool-hash）清单。
- 区分 Official MCP Registry 与各类元 registry（Glama、MCPMarket、MCP.so、Smithery、LobeHub）。

## 问题

一家《财富》500 强企业有 30 个已审批的 MCP 服务器、5000 名开发者、合规与审计要求，以及一个希望集中管理策略的安全团队。让每个开发者在自己的 IDE 里随意安装任意服务器是绝对不可接受的。

网关模式：

1. 网关作为单一的 Streamable HTTP 端点运行，开发者连接到它。
2. 网关持有每个后端 MCP 服务器的凭据。
3. 每个开发者的请求都通过网关自身的 OAuth 进行认证并限定范围。
4. 网关将调用路由到后端服务器，并施加策略。
5. 所有调用均被记录以供审计。

Cloudflare MCP Portals、Kong AI Gateway、IBM ContextForge、MintMCP、TrueFoundry、Envoy AI Gateway —— 它们都在 2025-2026 年推出了网关或网关相关功能。

与此同时，Official MCP Registry 作为规范上游上线：经过精选、命名空间验证、采用反向 DNS 命名的服务器，网关可以从中拉取。元 registry（Glama、MCPMarket、MCP.so、Smithery、LobeHub）则跨多个来源聚合服务器。

## 概念

### 网关的五项职责

1. **认证。** 用 OAuth 2.1 识别开发者；映射到用户角色。
2. **RBAC。** 按用户的策略：允许哪些服务器、哪些工具、哪些 scope。
3. **审计。** 每次调用都记录谁、做了什么、何时、结果如何。
4. **限流。** 按用户 / 按工具 / 按服务器设置上限，防止滥用。
5. **策略。** 拒绝被投毒的描述、强制执行 Rule of Two、脱敏 PII。

### 网关作为单一端点

在开发者看来，网关就像一个 MCP 服务器。其内部则路由到 N 个后端。会话 id（Phase 13 · 09）会在边界处被重写。

### 凭据保管（credential vaulting）

开发者永远看不到后端 token。网关持有这些 token（或代理到持有它们的身份提供方）。在网关上拥有 `notes:read` 的开发者，可以借助网关自身的后端凭据传递性地访问 notes MCP 服务器 —— 但只能在约束该传递访问的策略之下进行。

### 网关层的工具哈希固定（tool-hash pinning）

网关持有一份已审批工具描述的清单（SHA256 哈希）。在发现阶段，它会获取每个后端的 `tools/list`，将哈希与清单比对，并移除任何描述发生变更的工具。这是把 Phase 13 · 15 中的 rug-pull 防御集中化应用。

### 策略即代码（policy-as-code）

高级网关用 OPA/Rego、Kyverno 或 Styra 表达策略。诸如「用户 `alice` 仅可对 `acme` 组织下的仓库调用 `github.open_pr`」这样的规则被声明式地编码。简单网关则使用手写的 Python。两种形态都是有效的。

### 会话感知路由（session-aware routing）

当一个用户的会话包含多个服务器的混合时，网关进行多路复用：开发者的单个 MCP 会话持有 N 个后端会话，每个服务器一个。来自任意后端的通知都会经网关路由到开发者的会话。

### 命名空间合并

网关合并所有后端的工具命名空间，通常在冲突时加前缀。`github.open_pr`、`notes.search`。这使路由不产生歧义。

### Registries

- **Official MCP Registry（`registry.modelcontextprotocol.io`）。** 在 Anthropic、GitHub、PulseMCP、Microsoft 的共同维护下上线。经过命名空间验证（反向 DNS：`io.github.user/server`）。已针对基本质量做了预筛选。
- **Glama。** 以搜索为中心的元 registry，聚合众多来源。
- **MCPMarket。** 偏商业化的目录，带有厂商列表。
- **MCP.so。** 社区目录；开放提交。
- **Smithery。** 类似包管理器的安装流程。
- **LobeHub。** 集成在其 LobeChat 应用中的 UI 化 registry。

企业网关默认从 Official Registry 拉取，允许管理员从元 registry 精选添加，并拒绝任何未固定（unpinned）的内容。

### 反向 DNS 命名

Official Registry 要求公共服务器使用反向 DNS 命名：`io.github.alice/notes`。命名空间可防止抢注，并让信任委派更清晰。

### 厂商概览，2026 年 4 月

| 厂商 | 优势 |
|--------|------|
| Cloudflare MCP Portals | 边缘托管；集成 OAuth；有免费层 |
| Kong AI Gateway | K8s 原生；细粒度策略；日志输出到 OpenTelemetry |
| IBM ContextForge | 企业 IAM；合规；审计导出 |
| TrueFoundry | 偏 DevOps；指标优先 |
| MintMCP | 面向开发者平台 |
| Envoy AI Gateway | 开源；可定制过滤器 |

Phase 17（生产基础设施）会更深入地探讨网关运维。

## 动手用

`code/main.py` 用约 150 行实现了一个最小网关：通过一个假的 Bearer token 认证用户，持有按用户的 RBAC 策略，将请求路由到两个后端 MCP 服务器，将每次调用写入审计日志，强制执行限流，并拒绝任何描述哈希与固定清单不匹配的后端工具。

需要关注的地方：

- 以 `user_id` 为键的 `RBAC` 字典，其中包含允许的 `server_tool` 条目。
- `AUDIT_LOG` 是一个只追加（append-only）的事件列表。
- 限流使用按用户的令牌桶（token bucket）。
- 固定清单是一个 `server::tool -> hash` 的字典。

## 交付它

本课产出 `outputs/skill-gateway-bootstrap.md`。给定一份企业 MCP 方案（用户、后端、合规），该 skill 会生成一份网关配置规格。

## 练习

1. 运行 `code/main.py`。以一个被允许的用户发起调用；再以一个被拒绝的用户调用；然后发起一波超出限流的突发请求。验证这三种流程。

2. 添加一条策略，在将结果返回客户端之前对其中的 PII 做脱敏。使用一个简单的正则匹配 SSN 形态的字符串；指出其缺口（邮箱、电话号码）。

3. 扩展审计日志，使其发出 OpenTelemetry GenAI span。Phase 13 · 20 涵盖了确切的属性。

4. 为一个 50 人开发团队、五个后端（notes、github、postgres、jira、slack）设计一套 RBAC 策略。每个后端谁拥有只读权限？谁拥有写权限？

5. 从头到尾通读 Cloudflare 的企业 MCP 文章。找出一项 Cloudflare 提供而这个标准库网关没有的功能。

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|-----------|----------|
| Gateway | “MCP 代理” | 位于客户端与后端之间的集中化服务器 |
| Credential vaulting | “后端 token 留在服务器端” | 开发者永远看不到上游 token |
| Session-aware routing | “多后端会话” | 网关为每个开发者会话多路复用 N 个后端会话 |
| Tool-hash pinning | “已审批清单” | 每个已审批工具描述的 SHA256；集中阻止 rug-pull |
| RBAC | “按用户策略” | 针对工具和服务器的基于角色的访问控制 |
| Policy-as-code | “声明式规则” | 在网关强制执行的 OPA/Rego、Kyverno、Styra 策略 |
| Audit log | “谁、做了什么、何时” | 用于合规的只追加事件日志 |
| Rate limit | “按用户令牌桶” | 按分钟设置上限以防止滥用 |
| Official MCP Registry | “规范上游” | `registry.modelcontextprotocol.io`，经过命名空间验证 |
| Reverse-DNS naming | “registry 命名空间” | `io.github.user/server` 约定 |

## 延伸阅读

- [Official MCP Registry](https://registry.modelcontextprotocol.io/) —— 规范上游，经过命名空间验证
- [Cloudflare — Enterprise MCP](https://blog.cloudflare.com/enterprise-mcp/) —— 带 OAuth 与策略的网关模式
- [agentic-community — MCP gateway registry](https://github.com/agentic-community/mcp-gateway-registry) —— 开源参考网关
- [TrueFoundry — What is an MCP gateway?](https://www.truefoundry.com/blog/what-is-mcp-gateway) —— 功能对比文章
- [IBM — MCP context forge](https://github.com/IBM/mcp-context-forge) —— 来自 IBM 的企业网关
