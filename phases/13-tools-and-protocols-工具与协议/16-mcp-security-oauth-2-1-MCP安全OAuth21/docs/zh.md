# MCP Security II — OAuth 2.1、Resource Indicators、增量 Scope

> 远程 MCP 服务器需要的是授权（authorization），而不只是认证（authentication）。2025-11-25 规范与 OAuth 2.1 + PKCE + resource indicators（RFC 8707）+ protected-resource metadata（RFC 9728）保持一致。SEP-835 增加了增量 scope 同意机制，在收到 403 WWW-Authenticate 时进行 step-up 授权。本课将 step-up 流程实现为一个状态机，让你能看清每一跳。

**类型：** 构建
**语言：** Python（stdlib，OAuth 状态机模拟器）
**前置：** Phase 13 · 09（传输），Phase 13 · 15（安全 I）
**时长：** ~75 分钟

## 学习目标

- 区分资源服务器与授权服务器的职责。
- 走完受 PKCE 保护的 OAuth 2.1 授权码流程。
- 使用 `resource`（RFC 8707）和 protected-resource metadata（RFC 9728）防止 confused-deputy 攻击。
- 实现 step-up 授权：服务器返回 403 并带上 WWW-Authenticate 要求更高的 scope；客户端重新征求用户同意并重试。

## 问题

早期的 MCP（2025 年之前）在交付远程服务器时使用临时拼凑的 API key，甚至完全没有认证。2025-11-25 规范用完整的 OAuth 2.1 profile 弥补了这一缺口。

三个真实世界的需求：

- **普通远程服务器。** 用户安装一个访问其 Notion / GitHub / Gmail 的远程 MCP 服务器。带 PKCE 的 OAuth 2.1 是正确的形态。
- **Scope 升级。** 一个被授予 `notes:read` 的笔记服务器，之后可能需要 `notes:write` 来完成某个特定操作。step-up（SEP-835）只索取额外的 scope，而不是重做整个流程。
- **防止 confused deputy。** 客户端持有一个 audience 限定给服务器 A 的 token。服务器 A 是恶意的，试图将该 token 出示给服务器 B。Resource indicators（RFC 8707）将 token 固定在其预期的 audience 上。

OAuth 2.1 并不新鲜。新的是 MCP 的 profile：特定的必需流程（仅授权码 + PKCE；默认不允许 implicit，不允许 client credentials）、每次 token 请求都强制要求 resource indicators，以及发布 protected-resource metadata 以便客户端知道该去哪里。

## 概念

### 角色

- **客户端。** MCP 客户端（Claude Desktop、Cursor 等）。
- **资源服务器。** MCP 服务器（笔记、GitHub、Postgres，等等）。
- **授权服务器。** 颁发 token。可以与资源服务器是同一个服务，也可以是独立的 IdP（Auth0、Keycloak、Cognito）。

在 MCP 的 profile 中，资源服务器和授权服务器**可以**是同一台主机，但**应当**通过 URL 加以区分。

### 授权码 + PKCE

流程：

1. 客户端生成 `code_verifier`（随机）和 `code_challenge`（SHA256）。
2. 客户端将用户重定向到 `/authorize?response_type=code&client_id=...&redirect_uri=...&scope=notes:read&code_challenge=...&resource=https://notes.example.com`。
3. 用户同意。授权服务器重定向到 `redirect_uri?code=...`。
4. 客户端 POST 到 `/token?grant_type=authorization_code&code=...&code_verifier=...&resource=...`。
5. 授权服务器将 verifier 的哈希与存储的 challenge 进行校验，并颁发 access token。
6. 客户端使用该 token：每次向资源服务器发请求时携带 `Authorization: Bearer ...`。

PKCE 防止授权码拦截攻击。Resource indicators 防止 token 在别处也有效。

### Protected-resource metadata（RFC 9728）

资源服务器发布一份 `.well-known/oauth-protected-resource` 文档：

```json
{
  "resource": "https://notes.example.com",
  "authorization_servers": ["https://auth.example.com"],
  "scopes_supported": ["notes:read", "notes:write", "notes:delete"]
}
```

客户端从资源服务器发现授权服务器。这减少了配置——客户端只需要资源 URL。

### Resource indicators（RFC 8707）

token 请求中的 `resource` 参数将 token 的预期 audience 固定下来。所颁发的 token 包含 `aud: "https://notes.example.com"`。另一个收到该 token 的 MCP 服务器会检查 `aud` 并拒绝它。

### Scope 模型

Scope 是以空格分隔的字符串。常见的 MCP 约定：

- `notes:read`、`notes:write`、`notes:delete`
- `admin:*` 表示管理能力（谨慎使用）
- `profile:read` 表示身份

Scope 选择应遵循最小权限：现在需要什么就请求什么，需要更多时再 step up。

### Step-up 授权（SEP-835）

用户授予了 `notes:read`。他们之后请求智能体删除一条笔记。服务器响应：

```
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="insufficient_scope",
    scope="notes:delete", resource="https://notes.example.com"
```

客户端看到 insufficient_scope 错误，向用户弹出针对额外 scope 的同意对话框，为其执行一次小型 OAuth 流程，并用新 token 重试该请求。

### Token audience 校验

每次请求：服务器检查 `token.aud == self.resource_url`。不匹配 = 401。这阻止了跨服务器的 token 复用。

### 短期 token 与轮换

Access token **应当**是短期的（默认 1 小时）。Refresh token 在每次刷新时轮换。客户端在后台处理静默刷新。

### 不透传 token

Sampling 服务器（Phase 13 · 11）**禁止**将客户端的 token 透传给其他服务。Sampling 请求就是边界。

### 防止 confused deputy

Token 绑定到 `aud`。客户端绑定到 `client_id`。每次请求都同时对两者校验。规范明确禁止在 MCP 出现之前的远程工具生态中常见的旧式 "pass-the-token" 模式。

### Client ID 发现

每个 MCP 客户端在固定 URL 上发布其元数据。授权服务器可以拉取客户端的元数据文档，以发现 redirect URI 和联系信息。这消除了手动注册客户端的步骤。

### 网关与 OAuth

Phase 13 · 17 展示了企业网关如何处理 OAuth：网关持有上游服务器的凭据，颁发给客户端的 token 由网关签发，而上游 token 永远不离开网关。这翻转了信任模型——用户只需向网关认证一次；网关负责处理 N 个服务器的授权。

## 用一下

`code/main.py` 将完整的 OAuth 2.1 step-up 流程模拟为一个状态机。它实现了：

- PKCE code-verifier / challenge 生成。
- 带 resource indicator 的授权码流程。
- Protected-resource metadata 端点。
- 带 audience 检查的 token 校验。
- 在 `insufficient_scope` 时的 step-up。

本课不含 HTTP 服务器；状态机在内存中运行，让你可以追踪每一跳。Phase 13 · 17 的网关课会将其接入真实的传输。

## 交付

本课产出 `outputs/skill-oauth-scope-planner.md`。给定一个带工具的远程 MCP 服务器，该 skill 设计 scope 集合、固定（pinning）规则和 step-up 策略。

## 练习

1. 运行 `code/main.py`。追踪两个 scope 的 step-up 流程。注意 step-up 时哪些跳会重复。

2. 添加 refresh-token 轮换：每次刷新都颁发一个新的 refresh token 并使旧的失效。模拟一个被盗的 refresh token 在轮换后被使用，并确认它会失败。

3. 用 stdlib http.server 将 protected-resource metadata 端点实现为真实的 HTTP 响应。参照第 09 课的 /mcp 端点。

4. 为 GitHub MCP 服务器设计一个 scope 层级：read repo、write PR、approve PR、merge PR、admin。在每个层级之间使用 step-up。

5. 阅读 RFC 8707 和 RFC 9728。找出 9728 中那个 MCP 与 RFC 示例用法不同的字段。（提示：它涉及 `scopes_supported`。）

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| OAuth 2.1 | "现代 OAuth" | 整合后的 RFC，强制 PKCE 并禁止 implicit flow |
| PKCE | "持有证明（Proof-of-possession）" | code verifier + challenge，挫败授权码拦截 |
| Resource indicator | "Token audience" | RFC 8707 的 `resource` 参数，将 token 固定到单个服务器 |
| Protected-resource metadata | "发现文档" | RFC 9728 的 `.well-known/oauth-protected-resource` |
| Step-up 授权 | "增量同意" | SEP-835 流程，按需添加 scope |
| `insufficient_scope` | "带 WWW-Authenticate 的 403" | 服务器发出的信号，要求为更大的 scope 重新同意 |
| Confused deputy | "跨服务的 token 复用" | 一种攻击：受信任的持有者不当地转发 token |
| 短期 token | "Access token TTL" | 很快过期的 Bearer；由 refresh token 续期 |
| Scope 层级 | "最小权限栈" | 分级的 scope 集合，层级之间用 step-up |
| Client ID metadata | "客户端发现文档" | 客户端发布自身 OAuth 元数据的 URL |

## 延伸阅读

- [MCP — Authorization spec](https://modelcontextprotocol.io/specification/draft/basic/authorization) — 权威的 MCP OAuth profile
- [den.dev — MCP November authorization spec](https://den.dev/blog/mcp-november-authorization-spec/) — 2025-11-25 变更的逐步讲解
- [RFC 8707 — Resource indicators for OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc8707) — audience 固定的 RFC
- [RFC 9728 — OAuth 2.0 protected resource metadata](https://datatracker.ietf.org/doc/html/rfc9728) — 发现文档的 RFC
- [Aembit — MCP OAuth 2.1, PKCE and the future of AI authorization](https://aembit.io/blog/mcp-oauth-2-1-pkce-and-the-future-of-ai-authorization/) — 实用的 step-up 流程讲解
