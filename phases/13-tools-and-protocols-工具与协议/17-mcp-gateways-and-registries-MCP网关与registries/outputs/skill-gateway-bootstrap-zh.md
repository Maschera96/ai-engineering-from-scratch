---
name: gateway-bootstrap-zh
description: 在给定用户、后端和合规约束的情况下，产出一份网关配置规格。
version: 1.0.0
phase: 13
lesson: 17
tags: [mcp, gateway, rbac, audit, policy]
---

在给定一份企业级 MCP 方案（用户、后端、合规约束）的情况下，产出网关配置规格。

产出内容：

1. 后端列表。每个后端附带其 registry（Official / Glama / 自建）、规范名称（reverse-DNS）、固定（pinned）的描述哈希。
2. 用户列表。每个用户附带一个角色和允许使用的工具集。
3. RBAC 矩阵。每行对应一个用户 x 后端工具，标注 allow/deny。
4. 速率限制。每个用户的突发（burst）和持续（sustained）限制；对昂贵工具的按工具限制。
5. 审计方案。日志目的地（文件、OpenTelemetry、SIEM）、保留期、所采集的字段。

硬性拒绝项：
- 任何未在 Official Registry 中且未经管理员明确批准的后端。
- 任何允许所有用户使用所有工具的 RBAC 规则。会导致权限爆炸。
- 任何没有不可变存储的审计方案。合规失败。

拒绝规则：
- 如果开发者人数超过 100 且未定义任何角色，拒绝进行 bootstrap，并要求至少定义三个角色。
- 如果方案未指明 OAuth 2.1 身份提供方，拒绝并建议先采用 Keycloak 或 Auth0。
- 如果任何后端使用 stdio，拒绝通过 HTTP 网关代理它；stdio 服务器在每个开发者本地运行。

输出：一份一页纸的配置文档，包含后端列表、用户列表、RBAC 矩阵、速率限制和审计方案。结尾给出团队应当最先实施的那一条策略规则。
