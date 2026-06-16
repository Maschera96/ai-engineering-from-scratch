# Capstone 13 — 带 Registry 与治理能力的 MCP Server

> Model Context Protocol 在 2026 年不再只是未来，而成了默认的 tool-use 规范。Anthropic、OpenAI、Google 以及所有主流 IDE 都内置 MCP clients。Pinterest 发布了其内部 MCP server 生态。AAIF Registry 把 `.well-known` 上的 capability metadata 规范化了。AWS ECS 给出了无状态部署的参考方案。Block 的 goose-agent 则把同一协议放进了托管助手。2026 年的生产形态是，StreamableHTTP transport、OAuth 2.1 scopes、OPA policy gating，以及一个帮助平台团队发现、校验和启用 servers 的 registry。把整条链路端到端做出来。

**Type:** Capstone
**Languages:** Python (server, via FastMCP) or TypeScript (@modelcontextprotocol/sdk), Go (registry service)
**Prerequisites:** Phase 11 (LLM engineering), Phase 13 (tools and MCP), Phase 14 (agents), Phase 17 (infrastructure), Phase 18 (safety)
**Phases exercised:** P11 · P13 · P14 · P17 · P18
**Time:** 25 hours

## Problem

MCP 已经成了 tool-use 的通用语言。Claude Code、Cursor 3、Amp、OpenCode、Gemini CLI 以及所有托管型 agent，如今都在消费 MCP servers。难点不再是怎么写 server，FastMCP 已经让这件事足够简单，而是如何按企业要求在大规模环境中部署它们，包括按租户划分的 OAuth scopes、对破坏性 tools 的 OPA policy、支持水平扩展的 StreamableHTTP 无状态架构、用于发现能力的 registry，以及按每次 tool call 记录的 audit logs。Pinterest 的内部 MCP 生态和 AAIF Registry 规范定义了 2026 年的标准。

你将构建一个 MCP server，暴露 10 个内部 tools，覆盖 Postgres 只读、S3 listing、Jira、Linear、Datadog 等，再配一个用于平台发现的 registry UI，以及一个面向破坏性 tools 的人工审批 gate。负载测试需要展示 StreamableHTTP 的横向扩展能力。审计链路要满足企业安全审查要求。

## Concept

MCP 在 2026 年修订版中规定，StreamableHTTP 是默认 transport。与早期的 stdio + SSE 形态不同，StreamableHTTP 天生就是无状态的，一个 HTTP endpoint 接收 JSON-RPC requests，以流式返回 responses，并支持长连接通知。无状态意味着它可以轻松挂在负载均衡器后面做水平扩展。

授权使用带按 tool 细分 scopes 的 OAuth 2.1。一个 token 可以携带 `jira:read`、`s3:list`、`postgres:query:readonly` 之类的 scopes。MCP server 必须在每次 tool call 时检查 scopes，而不是只在 session 开始时检查。对高风险 tools，如果 scope 没有在最近 N 分钟内被提升到 `approved:by:human`，server 就要拒绝调用。这种 scope 提升来自一张 Slack review card。

Registry 是一个独立服务。每个 MCP server 都会暴露 `.well-known/mcp-capabilities` 文档，其中包含 tool manifest、transport URL 和 auth requirements。Registry 负责轮询、校验并建立索引。平台团队通过 registry UI 查看有哪些 tools 可用、需要哪些 scopes、由哪些团队负责。

## Architecture

```
MCP client (Claude Code, Cursor 3, ...)
          |
          v
StreamableHTTP over HTTPS (JSON-RPC + streaming)
          |
          v
MCP server (FastMCP) behind load balancer
          |
   +------+------+---------+----------+------------+
   v             v         v          v            v
Postgres    S3 listing  Jira       Linear     Datadog
(read-only) (paged)     (read)     (read)     (query)
          |
   +------+-------------+
   v                    v
 OPA policy gate   destructive tool MCP (separate server)
                        |
                        v
                   human approval via Slack
                        |
                        v
                   audit log (append-only, per-tenant)

  registry service
     |
     v  GET /.well-known/mcp-capabilities from each server
     v
     UI: search / validate / enable-disable / ownership
```

## Stack

- Server framework: FastMCP，Python，或 `@modelcontextprotocol/sdk`，TypeScript
- Transport: StreamableHTTP over HTTPS，无状态
- Auth: OAuth 2.1，配合 SPIFFE / SPIRE 提供 workload identity
- Policy: OPA / Rego rules，按 tool 细分，每次请求都要调用 policy decision service
- Registry: 自托管，消费 `.well-known/mcp-capabilities` manifests
- Human approval: 通过 Slack interactive message 审批破坏性 tools
- Deployment: AWS ECS Fargate 或 Fly.io，按租户独立部署或共享部署并带租户隔离
- Audit: 每租户一个结构化 JSONL bucket，记录每次调用的 lineage

## Build It

1. **Tool surface.** 暴露 10 个内部 tools，Postgres 只读查询、S3 列对象、Jira search/fetch、Linear search/fetch、Datadog metric query、PagerDuty 值班查询、GitHub 只读、Notion search、Slack search、Salesforce 只读。每个 tool 都有类型化 schema 和 scope label。

2. **FastMCP server.** 挂载这些 tools。配置 StreamableHTTP transport。增加一个 middleware，完成 OAuth token introspection 和 scope enforcement。

3. **OPA policy.** 为每个 tool 写 Rego policy，包含哪些 scopes 允许调用、需要怎样做 PII redaction、payload 大小上限是多少。每次 tool call 都要调用 decision service。

4. **Registry service.** 编写一个独立的 Go 或 TS 服务，轮询已注册 servers 的 `.well-known/mcp-capabilities`，用 JSON Schema 做校验，并提供 list / search / validate / enable-disable UI。

5. **Capability manifest.** 让每个 server 暴露 `.well-known/mcp-capabilities`，内容包括 tool list、auth requirements、transport URL、owner team 和 SLO。

6. **Destructive tool separation.** 会修改状态的 tools，比如 Jira create、Linear create、Postgres write，要放到第二个 MCP server 上，并使用更严格的 auth flow。Tokens 必须在 15 分钟内通过 Slack card 提升到 `approved:by:human` scope 才能调用。

7. **Audit log.** 每租户一份 append-only JSONL，记录 `{timestamp, user, tool, args_redacted, response_redacted, outcome}`。落盘前通过 Presidio 做 PII redaction。

8. **Load test.** 在 StreamableHTTP 上跑 100 个并发 clients。通过新增第二个 replica 展示水平扩展，并证明负载均衡器可以在无 session stickiness 的情况下重新分发流量。

9. **Conformance tests.** 对两个 servers 都运行官方 MCP conformance suite。通过所有 mandatory sections。

## Use It

```
$ curl -H "Authorization: Bearer eyJhbGc..." \
       -X POST https://mcp.internal.example.com/ \
       -d '{"jsonrpc":"2.0","method":"tools/call",
            "params":{"name":"postgres.readonly","arguments":{"sql":"SELECT 1"}}}'
[registry]   capability validated: postgres.readonly v1.2
[policy]    scope postgres:query:readonly present; allowed
[audit]     logged: user=u42 tool=postgres.readonly outcome=ok
response:    { "result": { "rows": [[1]] } }
```

## Ship It

`outputs/skill-mcp-server.md` 描述了可交付成果。它是一个面向内部 tools 的生产级 MCP server + registry + audit layer，带 OAuth 2.1 scopes 和 OPA gating。

| Weight | Criterion | How it is measured |
|:-:|---|---|
| 25 | Spec conformance | StreamableHTTP + capability manifest 通过 MCP conformance tests |
| 20 | Security | 对所有 tools 的 scope enforcement、OPA coverage 与 secret hygiene |
| 20 | Observability | 带 PII redaction 的逐 tool-call audit log |
| 20 | Scale | 100-client 负载测试与水平扩展示范 |
| 15 | Registry UX | Discover / validate / enable-disable workflow |
| **100** | | |

## Exercises

1. 增加一个新 tool，Confluence search。在不改动核心 server 的前提下，走完整个 registry validation flow。

2. 写一条 OPA policy，对 Postgres 查询结果中列名为 `email`、`ssn` 或 `phone` 的数据做 redaction。用一条 probe query 演练它。

3. Benchmark StreamableHTTP 与 stdio 的本地延迟差异。报告每次调用的 p50 / p95。

4. 实现按租户限额，每个租户每分钟每个 tool 最多 N 次调用。通过第二条 OPA rule 强制执行。

5. 运行 [mcp-conformance-tests](https://github.com/modelcontextprotocol/conformance) 中的 MCP conformance suite，并修复所有失败项。

## Key Terms

| Term | What people say | What it actually means |
|------|-----------------|------------------------|
| StreamableHTTP | “2026 MCP transport” | 无状态 HTTP + streaming，取代网络化 servers 中的 SSE + stdio |
| Capability manifest | “Well-known doc” | `.well-known/mcp-capabilities`，描述 tool list、auth、transport URL |
| OPA / Rego | “Policy engine” | 用 Open Policy Agent 按外部规则授权 tool calls |
| Scope elevation | “Approved-by-human” | 通过 Slack 审批获得的短期 scope，用于高风险 tools |
| Registry | “Tool discovery” | 根据 capability manifests 为 MCP servers 建立索引的服务 |
| Workload identity | “SPIFFE / SPIRE” | 用于签发 OAuth tokens 的密码学服务身份 |
| Conformance suite | “Spec tests” | 官方 MCP 测试集，用来验证 StreamableHTTP 和 tool manifest 的正确性 |

## Further Reading

- [Model Context Protocol 2026 Roadmap](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/) — StreamableHTTP、capability metadata、registry
- [AAIF MCP Registry spec](https://github.com/modelcontextprotocol/registry) — 2026 registry 规范
- [AWS ECS reference deployment](https://aws.amazon.com/blogs/containers/deploying-model-context-protocol-mcp-servers-on-amazon-ecs/) — 生产部署参考
- [Pinterest internal MCP ecosystem](https://www.infoq.com/news/2026/04/pinterest-mcp-ecosystem/) — 内部部署参考案例
- [Block `goose` MCP usage](https://block.github.io/goose/) — agent 消费模式参考
- [FastMCP](https://github.com/jlowin/fastmcp) — Python server framework
- [Open Policy Agent](https://www.openpolicyagent.org/) — policy engine 参考
- [SPIFFE / SPIRE](https://spiffe.io) — workload identity 参考
