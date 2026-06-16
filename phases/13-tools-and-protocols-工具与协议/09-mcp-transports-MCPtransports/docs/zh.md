# MCP Transports — stdio vs Streamable HTTP vs SSE 迁移

> stdio 只能在本地工作，在其他任何地方都不行。Streamable HTTP（2025-03-26）是远程标准。旧的 HTTP+SSE transport 已被弃用，将在 2026 年年中移除。选错 transport 要付出一次迁移的代价；选对则换来一个可远程托管的 MCP server，具备会话连续性和 DNS-rebinding 防护。

**类型：** 学习
**语言：** Python（stdlib，Streamable HTTP 端点骨架）
**前置：** Phase 13 · 07, 08（MCP server 与 client）
**时长：** ~45 分钟

## 学习目标

- 根据部署形态（本地 vs 远程、单进程 vs 集群）在 stdio 与 Streamable HTTP 之间做选择。
- 实现 Streamable HTTP 的单端点模式：POST 处理请求，GET 处理会话流。
- 强制执行 `Origin` 校验和 session-id 语义，以挫败 DNS-rebinding。
- 在 2026 年年中的移除截止期之前，将遗留的 HTTP+SSE server 迁移到 Streamable HTTP。

## 问题

MCP 的第一个远程 transport（2024-11）是 HTTP+SSE：两个端点，一个用于 client 的 POST，另一个是 Server-Sent-Events 通道，用于 server 到 client 的流。它能用，但也很笨拙：每个会话需要两个端点、在某些 CDN 前缓存会失效，并且强依赖长连接的 SSE，而某些 WAF 会激进地终止这类连接。

2025-03-26 规范用 Streamable HTTP 取代了它：单个端点，POST 处理 client 请求，GET 用于建立会话流，两者共享一个 `Mcp-Session-Id` 头。从那以后构建或迁移的每个 server 都使用 Streamable HTTP。旧的 SSE 模式正在被弃用——Atlassian Rovo 于 2026 年 6 月 30 日移除；Keboola 于 2026 年 4 月 1 日移除；其余大多数企业级 server 将在 2026 年底前移除。

而对于本地 server，stdio 仍然重要。Claude Desktop、VS Code 以及每个 IDE 形态的 client 都通过 stdio 启动 server。正确的心智模型是：stdio 用于「这台机器」，Streamable HTTP 用于「跨网络」。两者没有交叉。

## 概念

### stdio

- 子进程 transport。Client 启动 server，通过 stdin/stdout 通信。
- 每行一个 JSON 对象。以换行符分隔。
- 没有 session id；进程身份即会话。
- 无需鉴权（子进程继承父进程的信任边界）。
- 切勿用于远程 server——你将需要 SSH 或 socat 来做隧道，而那时还不如直接用 Streamable HTTP。

### Streamable HTTP

单个端点 `/mcp`（或任意路径）。支持三种 HTTP 方法：

- **POST /mcp。** Client 发送一条 JSON-RPC 消息。Server 回复单个 JSON 响应，或回复一个 SSE 流，其中包含一个或多个响应（适用于批量响应以及与该请求相关的通知）。
- **GET /mcp。** Client 打开一个长连接 SSE 通道。Server 用它发起 server 到 client 的请求（采样、通知、elicitation）。
- **DELETE /mcp。** Client 显式终止会话。

会话由 `Mcp-Session-Id` 头标识，server 在首个响应上设置该头，client 在之后的每个请求上回传它。Session id 必须是密码学随机的（128+ 位）；出于安全考虑，client 自选的 id 会被拒绝。

### 单端点 vs 双端点

旧规范的双端点模式在 2026 年仍可调用——规范将其声明为「遗留兼容」。但所有新 server 都应采用单端点。官方 SDK 输出单端点；仅在与未迁移的远程通信时才使用遗留模式。

### `Origin` 校验与 DNS-rebinding

浏览器（在今天）不是 MCP client，但攻击者可以精心构造一个网页，诱使浏览器向 `localhost:1234/mcp` 发起 POST——而用户的本地 MCP server 正监听在那里。如果 server 不检查 `Origin`，浏览器的同源策略救不了它，因为 `Origin: http://evil.com` 是合法的跨源请求。

2025-11-25 规范要求 server 拒绝 `Origin` 不在允许列表上的请求。允许列表通常包含 MCP client 主机（`https://claude.ai`、`vscode-webview://*`）以及供本地 UI 使用的 localhost 变体。

### Session id 生命周期

1. Client 发送首个请求，不带 `Mcp-Session-Id`。
2. Server 分配一个随机 id，在响应头上设置 `Mcp-Session-Id`。
3. Client 在之后的所有请求上、以及在用于流的 `GET /mcp` 上回传该头。
4. 会话可被 server 撤销；client 在后续请求上看到 404，必须重新初始化。
5. Client 可以显式 DELETE 会话以干净地关闭。

### 保活与重连

SSE 连接会断开。Client 通过用相同的 `Mcp-Session-Id` 重新 GET 来重新建立连接。Server 必须将中断期间错过的事件入队（在合理的窗口内），并通过 client 回传的 `last-event-id` 头进行重放。

Phase 13 · 13 涵盖 Tasks，它让长时间运行的工作即使在整个会话重连后也能存活下来。

### 向后兼容探测

一个想同时支持新旧 server 的 client：

1. POST 到 `/mcp`。
2. 如果响应是 `200 OK` 且带 JSON 或 SSE，这就是 Streamable HTTP。
3. 如果响应是 `200 OK` 且 `Content-Type: text/event-stream`，并带有一个指向次级端点的 `Location` 头，这就是遗留的 HTTP+SSE；跟随 `Location`。

### Cloudflare、ngrok 与托管

2026 年的生产级远程 MCP server 运行在 Cloudflare Workers（配合其 MCP Agents SDK）、Vercel Functions，或容器化的 Node/Python 上。关键：你的托管平台必须支持长连接的 HTTP，以便用于 SSE 的 GET。Vercel 免费层上限为 10 秒，不适用。Cloudflare Workers 支持无限期的流。

### 网关组合

当你用一个网关（Phase 13 · 17）置于多个 MCP server 之前时，该网关是一个单一的 Streamable HTTP 端点，它会重写 session id 并向上游做多路复用。工具在网关层合并；client 看到的是单个逻辑 server。

### Transport 失败模式

- **stdio SIGPIPE。** 子进程在写入中途死亡会引发 SIGPIPE；server 应干净地退出。Client 应检测 EOF 并将会话标记为已死。
- **HTTP 502 / 504。** Cloudflare、nginx 及其他代理在上游失败时会发出这些状态码。Streamable HTTP client 应在短暂退避后重试一次。
- **SSE 连接断开。** TCP RST、代理超时或 client 网络变更会关闭流。Client 用 `Mcp-Session-Id` 和可选的 `last-event-id` 重连以恢复。
- **会话撤销。** Server 使某个 session id 失效；client 在下一个请求上看到 404。Client 必须重新握手。
- **时钟偏移。** Client 上的资源 TTL 计算与 server 出现分歧。Client 应将 server 的时间戳视为权威。

### 何时绕过 Streamable HTTP

一些企业在自己的网络内部将 MCP server 部署在 gRPC 或消息队列 transport 之后。这是非标准做法——MCP 的规范并未正式定义这些。网关可以对 MCP client 暴露一个 Streamable HTTP 表面，而内部使用 gRPC。保持对外表面符合规范；由网关负责转换。

## 动手用

`code/main.py` 使用 `http.server`（stdlib）实现了一个最小的 Streamable HTTP 端点。它在 `/mcp` 上处理 POST、GET 和 DELETE，在首个响应上设置 `Mcp-Session-Id`，校验 `Origin`，并拒绝来自非允许列表来源的请求。该处理器复用了第 07 课 notes server 的分发逻辑。

要关注的地方：

- POST 处理器读取 JSON-RPC 主体、分发，并写出一个 JSON 响应（单响应变体；SSE 变体在结构上类似）。
- `Origin` 检查拒绝默认的 `http://evil.example` 探测，但接受 `http://localhost`。
- Session id 是随机的 128 位十六进制字符串；server 在内存中保存每个会话的状态。

## 交付

本课产出 `outputs/skill-mcp-transport-migrator.md`。给定一个 HTTP+SSE（遗留）MCP server，该 skill 生成一份迁移到 Streamable HTTP 的计划，包含 session-id 连续性、Origin 检查以及向后兼容的探测支持。

## 练习

1. 运行 `code/main.py`。用 `curl` POST 一个 `initialize`，观察 `Mcp-Session-Id` 响应头。POST 第二个请求并回传该头，验证会话连续性。

2. 添加一个打开 SSE 流的 GET 处理器。每五秒发送一个 `notifications/progress` 事件。用相同的 session id 重新 GET 来重连，并确认 server 接受它。

3. 实现 `last-event-id` 重放逻辑。重连时，重放自该 id 以来生成的所有事件。

4. 扩展 `Origin` 校验以支持通配符模式（`https://*.example.com`），并确认它接受 `https://app.example.com` 但拒绝 `https://evil.example.com.attacker.net`。

5. 从官方注册表中取一个遗留的 HTTP+SSE server（有好几个），勾勒出迁移方案：端点处理、session id 生成和头语义各有何变化。

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| stdio transport | 「本地子进程」 | 基于 stdin/stdout 的 JSON-RPC，以换行符分隔 |
| Streamable HTTP | 「远程 transport」 | 单端点 POST + GET + 可选 SSE，2025-03-26 规范 |
| HTTP+SSE | 「遗留」 | 将于 2026 年年中移除的双端点模型 |
| `Mcp-Session-Id` | 「会话头」 | Server 分配的随机 id，在之后的每个请求上回传 |
| `Origin` 允许列表 | 「DNS-rebinding 防御」 | 拒绝 Origin 未获批准的请求 |
| 单端点 | 「一个 URL」 | `/mcp` 处理所有会话操作的 POST / GET / DELETE |
| `last-event-id` | 「SSE 重放」 | 用于恢复断开的流而不丢失事件的头 |
| 向后兼容探测 | 「新旧检测」 | Client 通过响应形态检查自动选择 transport |
| 长连接 HTTP | 「SSE 流式」 | Server 在一个 TCP 连接上推送事件数分钟或数小时 |
| 会话撤销 | 「强制重新初始化」 | Server 使某个 session id 失效；client 必须重新握手 |

## 延伸阅读

- [MCP — Basic transports spec 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports) — stdio 与 Streamable HTTP 的权威参考
- [MCP — Basic transports spec 2025-03-26](https://modelcontextprotocol.io/specification/2025-03-26/basic/transports) — 引入 Streamable HTTP 的修订版
- [Cloudflare — MCP transport](https://developers.cloudflare.com/agents/model-context-protocol/transport/) — Workers 托管的 Streamable HTTP 模式
- [AWS — MCP transport mechanisms](https://builder.aws.com/content/35A0IphCeLvYzly9Sw40G1dVNzc/mcp-transport-mechanisms-stdio-vs-streamable-http) — 跨部署形态的对比
- [Atlassian — HTTP+SSE deprecation notice](https://community.atlassian.com/forums/Atlassian-Remote-MCP-Server/HTTP-SSE-Deprecation-Notice/ba-p/3205484) — 具体的迁移截止期示例
