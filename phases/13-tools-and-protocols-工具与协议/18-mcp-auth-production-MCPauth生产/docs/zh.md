# 生产环境中的 MCP 认证 —— 客户端注册、JWKS 刷新、受众绑定令牌

> 第 16 课在内存中搭建了 OAuth 2.1 状态机。到 2026 年，你交付给真实组织的每一个 MCP 服务器都位于生产级认证之后：可扩展到无上限客户端规模的客户端注册（首选 Client ID Metadata Documents，将动态客户端注册作为向后兼容的回退方案）、授权服务器元数据发现（RFC 8414 *或* OpenID Connect Discovery）、不会让凌晨三点的令牌校验崩溃的 JWKS 缓存刷新，以及拒绝跨资源重放的受众绑定令牌。本课用三个角色——授权服务器、资源服务器（即 MCP 服务器）和客户端——对整个面进行建模，让你能追踪从发现到一次通过校验的工具调用的每一跳。
>
> **规范说明（2025-11-25）：** 2025 年 11 月的 MCP 授权规范将动态客户端注册从 `SHOULD` 降级为 `MAY`，并将 **Client ID Metadata Documents（CIMD）** 设为推荐的默认注册机制。本课按规范的优先级顺序教授两者，而代码在演练中保留 DCR，因为它在单个进程内完全自包含。

**类型：** 构建
**语言：** Python（stdlib）
**前置：** Phase 13 · 16（OAuth 2.1 状态机）、Phase 13 · 17（网关）
**用时：** 约 90 分钟

## 学习目标

- 通过 RFC 8414 元数据发现授权服务器并校验其契约。
- 实现 RFC 7591 动态客户端注册，让 MCP 客户端无需管理员介入即可完成注册。
- 按计划缓存并刷新 JWKS 密钥，使签名校验在密钥轮换时仍能存活。
- 使用 RFC 8707 资源指示符将令牌绑定到单个 MCP 资源，并拒绝混淆代理式的复用。
- 干净地分离三个角色——授权服务器、资源服务器、客户端——使每个角色只执行属于自己的检查。
- 读懂 IdP 能力矩阵，当 IdP 无法满足 MCP 的认证档案时拒绝部署。

## 问题

第 16 课的模拟器在内存中运行 OAuth 2.1。生产环境有三个仅靠内存模拟器看不到的运维缺口。

第一个缺口是注册。真实组织运行数百个 MCP 服务器和数千个 MCP 客户端。运维人员不会把每个 Cursor 用户都手动注册为 OAuth 客户端。2025-11-25 规范给了客户端解决此问题的优先级顺序：如果你已有预注册的 `client_id` 就用它，否则使用 **Client ID Metadata Document**（客户端用自己掌控的一个 HTTPS URL 标识自身，授权服务器*拉取*该元数据），否则回退到 **RFC 7591 动态客户端注册**（客户端*推送*一次 `POST /register` 并当场获得一个 `client_id`），否则提示用户。CIMD 是推荐的默认方案，因为它在保持以 DNS 为根的信任模型的同时完全消除了逐服务器注册；DCR 则为向后兼容而保留。两者都从授权服务器的元数据中发现各自的入口：CIMD 对应 `client_id_metadata_document_supported`，DCR 对应 `registration_endpoint`。

第二个缺口是密钥轮换。JWT 校验依赖授权服务器的签名密钥，这些密钥以 JSON Web Key Set（JWKS）形式发布。授权服务器按计划轮换它们（通常每小时一次，在应急响应时有时更快）。一个在启动时只拉取一次 JWKS 的 MCP 服务器在轮换窗口前都能正常校验——而轮换之后，每个请求都会失败，直到重启。生产环境将 JWKS 接成一个缓存值，并配一个刷新作业，在上一批密钥过期前覆写缓存，再加上一个缓存未命中时的回退拉取，以应对收到由比缓存更新的密钥签名的令牌的情况。

第三个缺口是受众绑定。第 16 课引入了 RFC 8707 资源指示符。在生产环境中，该指示符在每个请求上变成一项硬性的声明检查。MCP 服务器将 `token.aud` 与自身的规范资源 URL 比较，对不匹配者以 HTTP 401 拒绝。这是抵御上游 MCP 服务器（或持有发给某一服务器的令牌的恶意客户端）将该令牌重放到同一信任网格中另一服务器的唯一防线。

本课将每个缺口映射到面上的一个具体部分。元数据文档是一个 HTTP 端点。JWKS 缓存刷新是一个计划作业加一个键值缓存。JWT 校验是资源服务器在分派任何工具前运行的一个例程。保持三个角色分离，每个角色只执行它拥有的检查：授权服务器签发并轮换密钥，资源服务器缓存并校验，客户端发现并注册。

## 概念

### RFC 8414 —— OAuth 授权服务器元数据

位于 `/.well-known/oauth-authorization-server` 的一个文档描述了客户端所需的一切：

```json
{
  "issuer": "https://auth.example.com",
  "authorization_endpoint": "https://auth.example.com/authorize",
  "token_endpoint": "https://auth.example.com/token",
  "jwks_uri": "https://auth.example.com/.well-known/jwks.json",
  "registration_endpoint": "https://auth.example.com/register",
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "code_challenge_methods_supported": ["S256"],
  "scopes_supported": ["mcp:tools.read", "mcp:tools.invoke"],
  "token_endpoint_auth_methods_supported": ["none", "private_key_jwt"]
}
```

拿到 MCP 资源 URL 的客户端会串联发现：RFC 9728 的 `oauth-protected-resource`（资源服务器的文档）给出 issuer，然后 `oauth-authorization-server`（本 RFC）给出每个端点。客户端从不硬编码授权 URL。

在信任某个 IdP 用于 MCP 之前你要校验的契约：

- `code_challenge_methods_supported` 包含 `S256`（按 RFC 7636 的 PKCE）。规范明确：如果该字段**缺失**，则授权服务器不支持 PKCE，客户端**必须**拒绝继续。
- `grant_types_supported` 包含 `authorization_code`，并拒绝 `password` 与 `implicit`。
- 至少公布一条注册路径：`client_id_metadata_document_supported: true`（CIMD，首选）**或** `registration_endpoint`（RFC 7591 DCR，回退）。任一即满足契约；你不再硬性要求 DCR。
- `response_types_supported` 对 OAuth 2.1 恰好为 `["code"]`。

如果缺少 `S256`，MCP 服务器拒绝部署到这个 IdP——PKCE 没有降级模式。如果*两条*注册路径都未公布且你没有预注册的 `client_id`，你同样无法注册；这时是部署清单有误，而非代码有误。

### RFC 9728（回顾）—— 受保护资源元数据

第 16 课讲过 RFC 9728。在生产环境中的差异：这个文档是客户端查找*本* MCP 服务器所信任的授权服务器的唯一处所。单个 MCP 服务器可以接受来自多个 IdP 的令牌（一个给员工，一个给合作伙伴）。RFC 9728 声明这个集合；RFC 8414 记录每个 IdP 支持什么。

```json
{
  "resource": "https://notes.example.com",
  "authorization_servers": ["https://auth.example.com", "https://partners.example.com"],
  "scopes_supported": ["mcp:tools.invoke"],
  "bearer_methods_supported": ["header"],
  "resource_documentation": "https://notes.example.com/docs"
}
```

### Client ID Metadata Documents（推荐的默认方案）

CIMD 把注册从*推送*翻转为*拉取*。客户端不再请求授权服务器铸造一个 `client_id`，而是把自己掌控的一个 HTTPS URL **当作**它的 `client_id`。该 URL 解析为一个 JSON 元数据文档；授权服务器在 OAuth 流程中按需拉取它。信任以 DNS 为根：如果服务器运营方信任 `app.example.com`，它就信任由 `https://app.example.com/client.json` 提供的客户端。没有注册往返、没有要耗尽的 `client_id` 命名空间、没有要逐服务器保持同步的状态。

客户端托管的元数据文档：

```json
{
  "client_id": "https://app.example.com/oauth/client.json",
  "client_name": "Example MCP Client",
  "client_uri": "https://app.example.com",
  "redirect_uris": ["http://127.0.0.1:7333/callback", "http://localhost:7333/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none"
}
```

文档中的 `client_id` 值**必须**等于其被托管的 URL（授权服务器会校验这一点；不匹配会被拒绝）。授权服务器在其 RFC 8414 元数据中以 `client_id_metadata_document_supported: true` 公布支持。

规范直言不讳的两条安全事实：

- **SSRF。** 授权服务器会拉取攻击者提供的 URL。它必须防御服务端请求伪造（不得拉取内部/管理端点）。
- **localhost 冒充。** 仅靠 CIMD 无法阻止本地攻击者声称某个合法客户端的元数据 URL 并绑定任意 `localhost` 回调。授权服务器**必须**在同意环节清晰展示回调 URI 的主机名，并**应当**对仅 `localhost` 的回调发出警告。

由于 CIMD 不需要服务端状态，就没有像 DCR 那样要搭建的注册器。客户端侧是只读的：从一个静态 HTTPS 端点提供你的元数据文档，让授权服务器去拉取它。

### RFC 7591 —— 动态客户端注册（回退 / 向后兼容）

DCR 现在是 `MAY`，为向后兼容 2025-11-25 之前的部署以及尚不支持 CIMD 的 IdP 而保留。没有它（且没有 CIMD 或预注册）的话，每个 MCP 客户端（Cursor、Claude Desktop、自定义智能体）都需要与 IdP 管理员进行带外交换。有了 DCR，客户端发送：

```json
POST /register
Content-Type: application/json

{
  "redirect_uris": ["http://127.0.0.1:7333/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none",
  "scope": "mcp:tools.invoke",
  "client_name": "Cursor",
  "software_id": "com.cursor.cursor",
  "software_version": "0.42.0"
}
```

服务器以 `client_id` 和一个供后续更新使用的 `registration_access_token` 响应：

```json
{
  "client_id": "c_3e7f1a",
  "client_id_issued_at": 1769472000,
  "redirect_uris": ["http://127.0.0.1:7333/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "registration_access_token": "regt_b2...",
  "registration_client_uri": "https://auth.example.com/register/c_3e7f1a"
}
```

对运行在用户设备上的 MCP 客户端来说，`token_endpoint_auth_method: none` 是正确的默认值。它们只拿到一个 `client_id`——没有可被窃取的 `client_secret`。PKCE 提供了公共客户端所需的持有证明。

三个生产陷阱：

- 注册端点必须按源 IP 限流。否则，敌对方会脚本化数百万次假注册并耗尽 `client_id` 命名空间。在注册器处理请求之前先运行一次限流检查。
- 某些企业 IdP 要求 `software_statement`（为客户端背书的已签名 JWT）。本课的 mock 跳过了它；生产环境会接一个校验步骤，对来自 localhost 回调 URI 以外任何来源的未签名注册予以拒绝。
- `registration_access_token` 必须以哈希形式存储，而非明文。盗取此令牌意味着攻击者可以改写客户端的回调 URI。

### RFC 8707（回顾）—— 资源指示符

第 16 课确立了其形态。生产规则：每个令牌请求都包含 `resource=<canonical-mcp-url>`，且 MCP 服务器在每次调用时校验 `token.aud` 是否与自身的资源 URL 匹配。规范 URI 是服务器*最具体*的标识符：它使用小写的 scheme 和 host、无 fragment，且按惯例无尾部斜杠。路径部分**不**按规则剥离——当需要它来标识某个具体 MCP 服务器时，规范会保留它。`https://mcp.example.com`、`https://mcp.example.com/mcp`、`https://mcp.example.com:8443` 和 `https://mcp.example.com/server/mcp` 都是有效的规范 URI。为每个服务器选一个，并把 `aud` 精确绑定到它。（本课的 mock 为简洁起见使用像 `https://notes.example.com` 这样的裸主机受众；在同一来源下共置多个 MCP 服务器的部署会通过路径区分它们。）

### RFC 7636（回顾）—— PKCE

PKCE 在 OAuth 2.1 中是强制的。本课的授权码流程始终携带 `code_challenge` 和 `code_verifier`。服务器拒绝任何不带 verifier、或 verifier 哈希后与存储的 challenge 不符的令牌请求。

### MCP 规范 2025-11-25 认证档案

MCP 规范（2025-11-25）对 MCP 服务器的授权层必须做什么有精确规定：

- 实现 RFC 9728 受保护资源元数据，并通过 401 上的 `WWW-Authenticate: Bearer resource_metadata="..."` 头部**或**众所周知 URI `/.well-known/oauth-protected-resource` 提供其位置（SEP-985 使该头部成为可选项，并带众所周知回退）。元数据的 `authorization_servers` 字段**必须**至少指明一个服务器。
- 仅通过**每个**请求上的 `Authorization: Bearer ...` 接受令牌——绝不放在查询字符串里，绝不只在会话开始时校验。
- 每个请求都校验 `aud`、`iss`、`exp` 和所需的作用域。服务器**必须**校验令牌是专门为它签发的（受众）；缺失或不匹配的 `aud` 会被拒绝，绝不当作通配符处理。
- 在 401/403 时，返回 `WWW-Authenticate: Bearer`，携带 `error=...`、`resource_metadata="<PRM-URL>"` 参数（元数据文档的 URL，*而非*裸资源），并在 `insufficient_scope`（403）时携带 `scope="..."`。注意：参数是 `resource_metadata`，一个发现指针——挑战中没有 `resource` 参数。
- 授权服务器发现接受 RFC 8414 OAuth 元数据**或** OpenID Connect Discovery 1.0 中的**任一**；客户端必须按优先级顺序尝试两个众所周知后缀。
- 客户端（而非服务器）防御**混淆攻击（mix-up attacks）**：它在重定向前记录预期的 `issuer`，并在兑换授权码前校验授权响应参数 `iss`（RFC 9207）。仅靠 PKCE 无法阻止 mix-up，因为客户端会把它的 `code_verifier` 交给它被引导到的任何令牌端点。

OAuth 2.1 草案是底层；RFC 8414/7591/8707/9728/9207 + RFC 7636 + CIMD 是面；MCP 规范是档案。

### IdP 能力矩阵

并非每个 IdP 都支持完整的 MCP 档案。下表记录了截至 2025-11-25 规范的事实性能力陈述。它是一道*部署门槛*，而非推荐。

CIMD 随 2025-11-25 规范一同发布，其底层 OAuth 草案直到 2025 年 10 月才被采纳，因此厂商支持仍在陆续到位——把下表中的"CIMD"理解为"当下的进展，请在你的租户中验证"，而非永久性陈述。

| IdP 类别 | AS 元数据（8414/OIDC） | CIMD | RFC 7591 DCR | RFC 8707 resource | RFC 7636 S256 PKCE | 说明 |
|---|---|---|---|---|---|---|
| 自托管（Keycloak） | 是 | 出现中 | 是 | 是（自 24.x 起） | 是 | 本课中 MCP 档案的参考 IdP；端到端完整 DCR 路径，CIMD 正跟进新规范。 |
| 企业 SSO（Microsoft Entra ID） | 是 | 出现中 | 是（高级层级） | 是 | 是 | DCR 可用性因租户层级而异；部署前请在目标租户中验证。 |
| 企业 SSO（Okta） | 是 | 出现中 | 是（Okta CIC / Auth0） | 是 | 是 | Auth0（现 Okta CIC）上提供 DCR；经典 Okta 组织需要管理员预注册。 |
| 社交登录 IdP（通用） | 视情况 | 否 | 罕见 | 罕见 | 是 | 多数社交 IdP 把客户端当作静态合作伙伴；无自助注册。仅作为身份来源使用，在其上层叠加你自己的、了解 MCP 的授权服务器。 |
| 自定义 / 自研 | 取决于 | 取决于 | 取决于 | 取决于 | 取决于 | 如果你自己交付，就交付完整档案并优先采用 CIMD。跳过 PKCE 或受众绑定会破坏 MCP 认证契约。 |

部署清单的拒绝规则：如果所选 IdP 未在 `code_challenge_methods_supported` 中列出 `S256`，MCP 服务器拒绝启动——PKCE 没有降级模式。注册是一道较宽松的门槛：你需要*一条*可用路径（预注册的 `client_id`、`client_id_metadata_document_supported: true`，或一个 `registration_endpoint`）。单单缺少 DCR 不再触发拒绝，因为 CIMD 或预注册可以覆盖它。

### JWKS 刷新模式（在 AS 处轮换，在资源服务器处刷新）

把两个动词分开，因为混为一谈是一个真实的生产 bug：

- **轮换（Rotate）**是*授权服务器*做的事：铸造一个新签名密钥，把它发布到 JWKS 中，稍后退役旧的。资源服务器在其中不起作用也无法做——它不持有 IdP 的私钥。
- **刷新（Refresh）**是*资源服务器*做的事：把已发布的 JWKS 重新 `GET` 到它的缓存中。这是资源服务器曾经执行的唯一 JWKS 操作。

生产环境的失败模式是缓存陈旧。用一个计划刷新作业加一个键值缓存来解决它。资源服务器运行一个作业（cron、定时器，或你的运行时所提供的任何东西），它以固定间隔拉取 `<issuer>/.well-known/jwks.json` 并覆写 `cache[issuer] = {keys, fetched_at}`。校验器从该缓存读取。一个 `kid` 在缓存中缺失的令牌会触发**一次**同步刷新作为回退，然后再次检查。这一次处理两种情况：计划刷新，以及由全新密钥签名的令牌在下次计划刷新前到达的密钥重叠窗口。

回退**必须是重新拉取，绝不是轮换**。如果你把缓存未命中路径接成轮换并铸造，会有两点崩溃：(1) 铸造一个新鲜密钥产生的 `kid` *仍然*与令牌不匹配，所以查找照样失败；(2) 一个用随机 `kid` 值喷洒令牌的攻击者会迫使产生一连串无上限的密钥创建——一次自伤式 DoS。重新拉取是幂等的，因此一个伪造的 `kid` 至多花费一次浪费的拉取。

缓存形态：

```json
{
  "https://auth.example.com": {
    "keys": [
      {"kid": "k_2026_03", "kty": "RSA", "n": "...", "e": "AQAB", "alg": "RS256", "use": "sig"},
      {"kid": "k_2026_04", "kty": "RSA", "n": "...", "e": "AQAB", "alg": "RS256", "use": "sig"}
    ],
    "fetched_at": 1772668800
  }
}
```

同时存在两个密钥是稳态。授权服务器轮换时先引入下一个密钥（`k_2026_04`），再退役上一个（`k_2026_03`），因此在旧密钥下签发的令牌在过期前仍然有效。缓存持有二者的并集；校验器按 `kid` 选取。

### 校验例程

MCP 服务器在分派任何工具前运行校验。`code/main.py` 使用的形态：

```python
result = server.validate(bearer_token, required_scope="mcp:tools.invoke")
if not result["valid"]:
    return {"status": result["status"], "WWW-Authenticate": result["www_authenticate"]}
```

`validate` 解码 JWT，从 JWKS 缓存解析签名密钥（未命中时刷新一次），校验签名，然后将 `iss` 与允许列表、`aud` 与本服务器的规范资源、`exp` 以及所需作用域逐一检查——在首个失败处返回一个 `WWW-Authenticate` 挑战。把它保持为资源服务器上的单个例程，意味着每个入口点（每次工具调用、每个传输）都经过同样的检查；没有任何路径能在未先校验的情况下到达工具。

### 受众重放演练（访问令牌权限限制）

服务器 A（`notes.example.com`）和服务器 B（`tasks.example.com`）都向同一个授权服务器注册。服务器 A 被攻陷。攻击者拿走某用户的 notes 令牌并把它重放到服务器 B。

服务器 B 的校验器：

1. 解码 JWT，按 `kid` 拉取 JWKS，校验签名。
2. 将 `iss` 与其受保护资源元数据的 `authorization_servers` 比对。（通过——同一 IdP。）
3. 检查 `aud == "https://tasks.example.com"`。（失败——令牌的 `aud` 是 `https://notes.example.com`。）
4. 返回 401，带 `WWW-Authenticate: Bearer error="invalid_token", error_description="audience mismatch", resource_metadata="https://tasks.example.com/.well-known/oauth-protected-resource"`。

受众声明是协议层抵御此攻击的唯一防线。为性能而跳过它是最常见的生产错误；校验器必须在每个请求上运行，而不只在会话开始时。规范称之为**访问令牌权限限制**：MCP 服务器**必须**拒绝任何未在受众中指明它的令牌。

> **命名说明。** 规范把术语*混淆代理（confused deputy）*保留给一个相关但不同的问题：一个充当对第三方 API 的 OAuth **代理**的 MCP 服务器，使用静态 client ID，在未获得逐客户端用户同意的情况下转发令牌。受众绑定修复上面的重放；混淆代理的修复是逐客户端同意**外加**绝不把入站令牌透传给上游 API（MCP 服务器**必须**获取它自己单独的上游令牌）。

### 混淆攻击（一种服务器无法提供的客户端侧防御）

一个客户端在其生命周期内与许多授权服务器对话。一个恶意 AS 可以试图让客户端在攻击者的令牌端点兑换一个诚实 AS 的授权码。受众绑定在这里帮不上忙——攻击发生在任何令牌存在之前。防御在客户端中（RFC 9207）：

1. 在重定向前，客户端从已校验的 AS 元数据中记录预期的 `issuer`。
2. 在授权响应上，客户端在把授权码发往任何地方之前，将返回的 `iss` 参数与所记录的 issuer 比对（简单字符串比较，不做规范化）。
3. 不匹配（或当 AS 公布了 `authorization_response_iss_parameter_supported` 时 `iss` 缺失）→ 拒绝，且连 `error` 字段都不显示。

仅靠 PKCE 无法阻止 mix-up，因为客户端会把它的 `code_verifier` 交给它被引导到的任何令牌端点。这就是为什么规范在 PKCE verifier 和 `state` 之外逐请求记录 issuer。

### 失败模式

- **JWKS 陈旧。** 校验器在 AS 轮换某个密钥后拒绝有效令牌。修复是上面的 cron 刷新 + 缓存未命中重拉模式。绝不在没有刷新作业的情况下缓存 JWKS。
- **以轮换作回退。** 把缓存未命中路径接成轮换并铸造而非重新拉取是一个真实的 bug：它永远产生不出缺失的 `kid`，并且把攻击者控制的 `kid` 值变成密钥创建 DoS。回退必须是幂等的 `refresh-jwks`。
- **缺失 `aud` 声明。** 某些 IdP 默认在令牌请求中没有 `resource` 时省略 `aud`。校验器必须拒绝缺失 `aud` 的令牌，而不能把缺失当作通配符。
- **因缺失 `iss` 检查导致的 mix-up。** 一个不把 RFC 9207 `iss` 授权响应参数与重定向前所记录 issuer 校验的客户端，可能被引导到在攻击者的令牌端点兑换诚实 AS 的授权码。这是客户端侧的失败；资源服务器无法弥补它。
- **作用域升级竞态。** 同一用户的两个并发升级流程可能都成功，并产生两个作用域不同的访问令牌。校验器必须使用请求上呈递的令牌，而不是查找"用户当前的作用域"——后者会制造一个 TOCTOU 窗口。
- **注册令牌被盗。** 泄露的 `registration_access_token` 让攻击者得以改写回调 URI。把它们以哈希形式静态存储；要求客户端在每次更新时呈递明文；在可疑时轮换。
- **`iss` 未绑定。** 一个接受任意 `iss` 的校验器让攻击者得以搭建他们自己的授权服务器，为目标受众注册一个客户端，并签发令牌。受保护资源元数据的 `authorization_servers` 列表是允许列表；强制执行它。

## 使用它

`code/main.py` 用 stdlib Python 和三个角色——`AuthorizationServer`、`ResourceServer` 和 `Client`——走完整个生产流程。流程：

1. 授权服务器在 `/.well-known/oauth-authorization-server` 发布 RFC 8414 元数据。
2. MCP 客户端调用元数据端点，检查其注册选项（CIMD 对应 `client_id_metadata_document_supported`，DCR 对应 `registration_endpoint`）和 `S256` PKCE 支持。
3. 演练走 DCR 回退路径：客户端向 `/register` 发送请求（RFC 7591）并收到一个 `client_id`。（CIMD 客户端会改为呈递自己的 HTTPS `client_id` URL 并跳过此步。）
4. MCP 客户端运行带 PKCE 保护的授权码流程（RFC 7636），携带 `resource` 指示符（RFC 8707）。
5. MCP 客户端用 `Authorization: Bearer ...` 调用 MCP 服务器上的一个工具。
6. MCP 服务器运行 `validate`，从 JWKS 缓存解析签名密钥。
7. IdP 轮换一个密钥；计划刷新把 JWKS 重新拉取到缓存。
8. 下一次调用无需重启即可针对刷新后的密钥校验，且上一个令牌在重叠窗口内仍能校验通过。
9. 针对另一个 MCP 资源的受众重放尝试得到带 `audience mismatch` 和一个 `resource_metadata` 指针的 401。

这里的 JWT 使用带共享密钥的 HS256（以便本课仅靠 stdlib 运行）。生产环境采用上面的 JWKS 模式配合 RS256 或 EdDSA；校验逻辑在其他方面完全相同。由于 IdP 和资源服务器位于同一进程，`refresh_jwks` 直接读取授权服务器的密钥列表；在网络上它是对 `jwks_uri` 的一次 HTTP `GET`。

## 交付它

本课产出 `outputs/skill-mcp-auth.md`。给定一个 MCP 服务器配置和一组 IdP 能力，该 skill 输出要搭建的认证面——受保护资源元数据、要使用的注册路径（CIMD、预注册或 DCR 回退）、JWKS 刷新计划、作用域映射，以及当 IdP 不支持完整 RFC 档案时要应用的拒绝规则。

## 练习

1. 运行 `code/main.py`。追踪流程。注意 IdP 如何在第 6 步轮换一个密钥，计划的 `refresh_jwks` 重新拉取已发布的集合，以及旧令牌（重叠窗口）和新令牌如何无需重启都校验通过。

2. 向受保护资源元数据的 `authorization_servers` 列表添加一个新 IdP。签发一个由新 IdP 签名的令牌并确认校验器接受它。签发一个由未列出的 IdP 签名的令牌并确认校验器以 `WWW-Authenticate: Bearer error="invalid_token", error_description="iss not allowed"` 拒绝。

3. 给 `register_client` 加一个在注册器接受请求之前运行的限流检查。使用一个按源 IP 的令牌桶，存放在以 IP 为键的小字典中。

4. 阅读 RFC 7591 并找出本课 `/register` 处理器未校验的两个字段。添加校验。（提示：`software_statement` 和 `redirect_uris` URI scheme。）

5. 添加一条 Client ID Metadata Document 路径。提供一个 `client_id` 等于自身 URL 的 `client.json`，并让授权服务器拉取并校验它（若 `client_id` ≠ URL 则拒绝）。确认一个 CIMD 客户端在没有 `register_client` 调用的情况下完成注册。

6. 证明 DoS 修复。给校验器发送一个带随机 `kid` 的令牌并确认 `refresh_jwks` 至多运行一次且授权服务器的密钥数量不增长。然后故意把回退重新接成轮换并铸造，观察密钥数量随每个伪造令牌攀升——之后恢复重新拉取。

7. 实现 mix-up 一节中的客户端侧 RFC 9207 `iss` 检查：在授权请求前记录预期的 issuer，然后拒绝其 `iss` 不匹配的授权响应。

## 关键术语

| 术语 | 人们怎么说 | 它实际指什么 |
|------|----------------|------------------------|
| ASM | "OAuth 元数据文档" | RFC 8414 `/.well-known/oauth-authorization-server` JSON |
| CIMD | "客户端元数据 URL" | Client ID Metadata Document——一个被用作 `client_id` 的 HTTPS URL；AS 拉取该 JSON。自 2025-11-25 起为推荐默认方案 |
| DCR | "自助式客户端注册" | RFC 7591 `POST /register` 流程；在 2025-11-25 中降级为 `MAY` 回退 |
| JWKS | "用于 JWT 校验的公钥" | JSON Web Key Set，从 `jwks_uri` 拉取，按 `kid` 索引 |
| 轮换 vs 刷新 | "更新密钥" | *轮换* = AS 铸造/退役签名密钥；*刷新* = 资源服务器重新拉取已发布集合。资源服务器只做刷新 |
| 资源指示符 | "受众参数" | RFC 8707 `resource` 参数，把令牌绑定到一个服务器 |
| `aud` 声明 | "受众" | 校验器与规范资源 URL 比对的 JWT 声明 |
| 受众重放 | "令牌重放" | 为服务器 A 签发的令牌被呈递给服务器 B；通过受众校验防御（规范：访问令牌权限限制） |
| 混淆代理 | "代理令牌误用" | 一个带静态 client ID 的 MCP 代理在无逐客户端同意下转发令牌；与受众重放不同 |
| 混淆攻击 | "错误的令牌端点" | 客户端被引导到在攻击者端点兑换诚实 AS 的授权码；通过 RFC 9207 `iss` 在客户端侧防御 |
| `iss` 允许列表 | "受信任的授权服务器" | 受保护资源元数据的 `authorization_servers` 中指明的集合 |
| `resource_metadata` | "去哪里找 PRM 文档" | 401/403 上 `WWW-Authenticate` 参数，指明 RFC 9728 元数据 URL |
| 公共客户端 | "原生或浏览器客户端" | 无 `client_secret` 的 OAuth 客户端；由 PKCE 弥补 |
| `WWW-Authenticate` | "401/403 响应头" | 携带驱动客户端恢复的 `Bearer error=...` 指令 |

## 延伸阅读

- [MCP —— 授权规范（2025-11-25）](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization) —— 本课实现的 MCP 认证档案
- [MCP 博客 —— MCP 周年：2025 年 11 月规范发布](https://blog.modelcontextprotocol.io/posts/2025-11-25-first-mcp-anniversary/) —— 2025-11-25 中的变化（CIMD、XAA、DCR 降级）
- [Aaron Parecki —— 2025 年 11 月 MCP 授权规范中的客户端注册](https://aaronparecki.com/2025/11/25/1/mcp-authorization-spec-update) —— CIMD 优先于 DCR 的理由
- [OAuth Client ID Metadata Document（draft-ietf-oauth-client-id-metadata-document-00）](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-client-id-metadata-document-00) —— CIMD
- [RFC 8414 —— OAuth 2.0 Authorization Server Metadata](https://datatracker.ietf.org/doc/html/rfc8414) —— 发现契约
- [RFC 7591 —— OAuth 2.0 Dynamic Client Registration Protocol](https://datatracker.ietf.org/doc/html/rfc7591) —— DCR（回退路径）
- [RFC 7636 —— Proof Key for Code Exchange (PKCE)](https://datatracker.ietf.org/doc/html/rfc7636) —— 公共客户端持有证明
- [RFC 8707 —— Resource Indicators for OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc8707) —— 受众绑定
- [RFC 9728 —— OAuth 2.0 Protected Resource Metadata](https://datatracker.ietf.org/doc/html/rfc9728) —— 资源服务器发现
- [RFC 9207 —— OAuth 2.0 Authorization Server Issuer Identification](https://datatracker.ietf.org/doc/html/rfc9207) —— 防御 mix-up 攻击的 `iss` 参数
- [OAuth 2.1 草案](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1) —— 整合后的 OAuth 底层
