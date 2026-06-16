---
name: mcp-server-platform-zh
description: 部署生产级 MCP server，具备 StreamableHTTP、OAuth 2.1 scopes、OPA policy、破坏性工具的人类 approval gate，以及用于 discovery 的 registry。
version: 1.0.0
phase: 19
lesson: 13
tags: [capstone, mcp, fastmcp, streamablehttp, oauth, opa, registry, governance]
---

给定一个企业环境，交付一个带 10 个内部工具的 MCP server、一个用于 discovery 的 registry service，以及一个通过 Slack approval gate 破坏性工具的 governance layer。

构建计划：

1. FastMCP server 暴露 10 个只读工具（Postgres、S3、Jira、Linear、Datadog、PagerDuty、GitHub、Notion、Slack、Salesforce），每个工具都有 typed schema 和 required scope。
2. StreamableHTTP transport，在 load balancer 后保持 stateless。
3. OAuth 2.1 token introspection middleware；workload identity 使用 SPIFFE / SPIRE。
4. 每个 tool call 都经过 OPA / Rego policy decisions：scope enforcement、PII redaction、payload size caps。
5. 破坏性工具（Jira create、Linear create、Postgres write）放在单独 MCP server 上，需要 scope `approved:by:human`，该 scope 通过 15 分钟内有效的 Slack card 提权获得。
6. Registry service 轮询每个 server 的 `.well-known/mcp-capabilities`，用 JSON Schema 验证，并暴露 list/search/validate/enable UI。
7. 每 tenant 一个 JSONL audit log，写入前用 Presidio PII redaction。
8. 100-client load test 证明横向扩展；通过 MCP conformance suite。

评估标准：

| Weight | Criterion | Measurement |
|:-:|---|---|
| 25 | Spec conformance | StreamableHTTP + capability manifest 通过 MCP conformance tests |
| 20 | Security | Scope enforcement、所有 tools 上的 OPA coverage、secret hygiene |
| 20 | Observability | 每个 tool-call audit log 写入时做 PII redaction |
| 20 | Scale | 100-client load test，并展示 horizontal scale |
| 15 | Registry UX | Discover / validate / enable-disable workflow 被演练 |

硬性拒收：

- 需要 stateful sessions 的 servers（违反 2026 StreamableHTTP stateless contract）。
- 破坏性工具与只读工具共享同一 auth surface 的单 server topology。
- 持久化 raw PII 的 audit logs。
- 忽略 capability manifest；registry integration 是硬要求。

拒绝规则：

- 没有 OAuth 时拒绝部署；匿名访问直接淘汰。
- 没有 Slack approval flow 时拒绝发布破坏性工具。
- 工具的 scope 或 description 不在 capability manifest 中时，拒绝暴露该工具。

输出：一个仓库，包含两个 MCP servers（read-only + destructive）、registry service、Slack approval integration、OPA policies、100-client load-test harness、conformance-test results，以及一份说明文档，描述你考虑过但没有暴露的工具（以及原因），再列出 dry-run 中捕获 near-misses 的前三条 OPA rules。
