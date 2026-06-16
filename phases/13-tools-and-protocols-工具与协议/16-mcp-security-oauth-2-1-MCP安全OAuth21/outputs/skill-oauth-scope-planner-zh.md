---
name: oauth-scope-planner-zh
description: 为远程 MCP 服务器设计 OAuth 2.1 的 scope 集合、绑定规则与升级（step-up）策略。
version: 1.0.0
phase: 13
lesson: 16
tags: [oauth, pkce, resource-indicators, step-up, sep-835]
---

给定一个带有工具列表的远程 MCP 服务器，为其设计授权模型。

产出：

1. Scope 层级。分级的 scope 集合（例如 `read` -> `write` -> `delete` -> `admin`）。每个操作类别对应一个 scope；不要让 scope 集合膨胀。
2. Scope 与工具的映射。为每个工具标注其所需的 scope。标记出任何需要多个 scope 的工具。
3. 升级策略。哪些操作需要升级（step-up）而非初始同意。典型情况：破坏性操作需要升级。
4. 资源指示符（resource indicator）值。`resource` 参数中使用的规范 URL。确保该 URL 与 `.well-known/oauth-protected-resource` 的 resource 字段一致。
5. 受保护资源元数据。起草包含 `authorization_servers`、`scopes_supported` 和 `resource` 的 `.well-known/oauth-protected-resource` JSON。

硬性拒绝项：
- 任何需要 admin scope 但在调用时没有明确确认对话框的工具。需要升级。
- 任何覆盖多个操作类别的 scope。权限蔓延（privilege creep）。
- 任何跳过受众（audience）校验的服务器。混淆代理（confused-deputy）漏洞。

拒绝规则：
- 如果服务器是本地（stdio）的，拒绝 OAuth，并说明 stdio 继承父进程的信任。
- 如果服务器依赖遗留的 OAuth 2.0 隐式流（implicit flow），拒绝并强制迁移到 2.1 + PKCE。
- 如果用户要求无密码的“仅 API key”认证，对远程服务器予以拒绝；要求使用 OAuth 2.1 授权码 + PKCE 加资源指示符来实现用户授权的访问。客户端凭据（client credentials）仅适用于无用户委托的机器对机器场景。

输出：一页授权计划，包含 scope 层级、scope 与工具的映射、升级策略、资源指示符，以及受保护资源元数据 JSON。最后给出最可能让用户在首次遇到时感到意外的升级操作。
