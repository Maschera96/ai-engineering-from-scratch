---
name: mcp-auth-wiring-zh
description: 搭建生产级 MCP 授权（RFC 8414、CIMD、7591、8707、7636 PKCE、9728、9207）——受保护资源元数据、注册登记、JWKS 刷新以及每请求令牌校验。
version: 1.1.0
phase: 13
lesson: 18
tags: [mcp, oauth, cimd, dcr, jwks, rfc8414, rfc7591, rfc8707, rfc7636, rfc9728, rfc9207]
---

给定一份 MCP 服务器配置和一套 IdP 能力集，输出构成生产级 MCP 授权层的认证面与拒绝规则。

输入：

- `mcp_resource_url` —— 规范资源 URL（最具体的标识符；仅当用于区分同主机托管的多个服务器时才保留路径），用作 `aud` 以及受保护资源元数据中的 `resource` 值。
- `idp_metadata_url` —— IdP 的 `/.well-known/oauth-authorization-server`（或 OpenID Connect Discovery）URL。
- `idp_capabilities` —— 观察到的 `code_challenge_methods_supported`、`grant_types_supported`、`client_id_metadata_document_supported`（CIMD）、`registration_endpoint`（DCR）、`response_types_supported`、`authorization_response_iss_parameter_supported`（RFC 9207）的取值。
- `tools` —— MCP 工具列表，附带每个工具所需的 scope。

产出：

1. **拒绝关口（Refusal gate）。** 若以下任一硬性条件不满足，则拒绝接线并停止：
   - `code_challenge_methods_supported` 中缺少 `S256`（PKCE 没有降级模式）。
   - `grant_types_supported` 中缺少 `authorization_code`。
   - `response_types_supported` 不是恰好为 `["code"]`。
   - 不存在任何注册登记路径：预注册的 `client_id`、`client_id_metadata_document_supported: true`（CIMD）、`registration_endpoint`（DCR）三者皆无。三者任一即可——单凭缺少 DCR 不再构成拒绝（2025-11-25 将 DCR 降级为 `MAY`；CIMD 是首选默认项）。

2. **受保护资源元数据文档**（RFC 9728），由 MCP 服务器发布于 `/.well-known/oauth-protected-resource`。包含 `resource`、`authorization_servers`（签发方允许列表）、`scopes_supported`、`bearer_methods_supported: ["header"]`。

3. **HTTP 端点。**
   - `GET /.well-known/oauth-protected-resource` —— 返回 (2) 中的文档。
   - `POST /mcp`（MCP 传输层）—— 在任何工具分发之前先运行令牌校验。
   - （仅 DCR 路径）`POST /register` —— 注册器，其前置一道限流检查。

4. **后台任务 + 例程。**
   - 一个定时 JWKS 刷新任务，将 `jwks_uri` 重新拉取到缓存 `{keys, fetched_at}`。幂等；绝不铸造密钥。AS 负责轮换，资源服务器只负责刷新。默认 `0 */6 * * *`；对高轮换频率的 IdP 收紧为 `*/15 * * * *`。
   - 一个 `validate` 例程 —— 校验 `iss` 允许列表、对照缓存 JWKS 的签名、`aud == mcp_resource_url`、`exp`、所需 scope。
   - 一条 step-up 签发路径 —— 仅当工具列表包含被某个用户初始未授予的 scope 所门禁的操作时才需要。

5. **缓存方案。** 每个被接受的签发方一条条目，以 `issuer` 为键，保存 `{keys, fetched_at}`。记录读取模式：校验器读取缓存，在 `kid` 未命中时回退到一次同步刷新（重新拉取，而非轮换——重新拉取是幂等的，无法被转化为创建密钥的 DoS）。

6. **Scope 映射。** 将每个工具映射到其所需的 scope。输出一张表：
   `| tool | required_scope | rationale |`。将破坏性工具归入它们各自专属的 scope；切勿为写操作工具复用读取 scope。

7. **运行时拒绝规则**（校验器必须编码这些规则）：
   - 当 `aud != mcp_resource_url` 时拒绝 → 401 `Bearer error="invalid_token", error_description="audience mismatch", resource_metadata="<prm_url>"`。
   - 当 `iss not in authorization_servers` 时拒绝。
   - 当经过一次重新拉取回退后 `kid` 仍不在缓存 JWKS 中时拒绝。
   - 当所需 scope 缺失时拒绝 → 403 `Bearer error="insufficient_scope", scope="<required>", resource_metadata="<prm_url>"`。
   - 拒绝任何缺少 `code_verifier` 或 `resource` 参数的令牌请求。

硬性拒绝（以下任何一项都绝不接线——拒绝该请求并记录原因）：

- 以明文存储 `client_secret`。公开客户端使用 `token_endpoint_auth_method: none`；机密客户端使用 `private_key_jwt`。静态存储或注册响应日志中都不得出现明文共享密钥。
- 在校验器上跳过 `aud` 检查。受众绑定（访问令牌权限限制）正是 RFC 8707 + RFC 9728 存在的全部理由。
- 将 JWKS 缓存未命中回退接线成「轮换并铸造」而非「重新拉取」。前者永远产生不出缺失的 `kid`，并让攻击者控制的 `kid` 值得以强制无界地创建密钥。该回退必须是幂等的刷新。
- 允许无 PKCE 的授权码请求。OAuth 2.1 禁止这样做；校验器必须拒绝任何其所存授权码记录缺少 `code_challenge` 的 `/token` 交换。
- 缓存 JWKS 却没有刷新任务。要么定时刷新随之上线，要么这套认证面不予部署。
- 在没有允许列表的情况下信任 `iss` 声明。任何接受来自任意 `iss` 的令牌的校验器，都会让攻击者得以架设自己的 IdP 并伪造令牌。
- 将入站的 MCP 令牌转发给上游 API（令牌透传）。如果 MCP 服务器调用上游 API，它必须获取自己独立的令牌；透传会制造混淆代理（confused-deputy）问题。
- 以明文存储 `registration_access_token`。静态存储须哈希化；每次更新都要求提供明文。

输出：一页式方案，包含受保护资源文档、所选注册登记路径（CIMD / 预注册 / DCR）、HTTP 端点、JWKS 刷新任务、缓存方案、scope 映射表，以及编码后的运行时拒绝规则。以最可能在所选 IdP 上暴露出来的那一个阻断部署的缺口收尾——通常是 CIMD 是否已受支持，并退而判断面向企业 SSO 的 DCR 可用性。
