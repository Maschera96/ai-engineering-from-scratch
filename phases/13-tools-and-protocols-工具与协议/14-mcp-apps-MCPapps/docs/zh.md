# MCP Apps — 通过 `ui://` 实现交互式 UI 资源

> 纯文本的工具输出限制了智能体的展示能力。MCP Apps（SEP-1724，2026 年 1 月 26 日正式发布）让工具能够返回沙箱化的交互式 HTML，并在 Claude Desktop、ChatGPT、Cursor、Goose 和 VS Code 中内联渲染。仪表盘、表单、地图、3D 场景，全都通过一个扩展实现。本课讲解 `ui://` 资源方案、`text/html;profile=mcp-app` MIME、iframe 沙箱的 postMessage 协议，以及让服务器渲染 HTML 所带来的安全面。

**类型：** Build
**语言：** Python（标准库，UI 资源发射器）、HTML（示例应用）
**前置要求：** Phase 13 · 07（MCP 服务器）、Phase 13 · 10（资源）
**时长：** 约 75 分钟

## 学习目标

- 从工具调用中返回一个 `ui://` 资源，并设置正确的 MIME 和元数据。
- 通过 `_meta.ui.resourceUri`、`_meta.ui.csp` 和 `_meta.ui.permissions` 声明工具关联的 UI。
- 实现用于 UI 到宿主通信的 iframe 沙箱 postMessage JSON-RPC。
- 应用 CSP 和 permissions-policy 默认值，防御源自 UI 的攻击。

## 问题所在

一个 2025 年代的 `visualize_timeline` 工具可以返回"这是按时间顺序组织的 14 条笔记：……"。那只是一段文字。用户真正想要的是交互式时间线。在 MCP Apps 出现之前，选项只有：客户端专属的小部件 API（Claude artifacts、OpenAI Custom GPT HTML），或者干脆没有 UI。

MCP Apps（SEP-1724，于 2026 年 1 月 26 日发布）将这一契约标准化。工具结果包含一个 `resource`，其 URI 为 `ui://...`，MIME 为 `text/html;profile=mcp-app`。宿主在一个沙箱化的 iframe 中渲染它，采用受限的 CSP，且除非显式授权否则没有网络访问。iframe 内的 UI 通过一种小巧的 postMessage JSON-RPC 方言向宿主发送消息。

每个兼容客户端（Claude Desktop、ChatGPT、Goose、VS Code）都以相同方式渲染同一个 `ui://` 资源。一个服务器、一个 HTML 包、通用的 UI。

## 核心概念

### `ui://` 资源方案

工具返回：

```json
{
  "content": [
    {"type": "text", "text": "Here is your notes timeline:"},
    {"type": "ui_resource", "uri": "ui://notes/timeline"}
  ],
  "_meta": {
    "ui": {
      "resourceUri": "ui://notes/timeline",
      "csp": {
        "defaultSrc": "'self'",
        "scriptSrc": "'self' 'unsafe-inline'",
        "connectSrc": "'self'"
      },
      "permissions": []
    }
  }
}
```

随后宿主对 `ui://notes/timeline` URI 调用 `resources/read`，得到返回：

```json
{
  "contents": [{
    "uri": "ui://notes/timeline",
    "mimeType": "text/html;profile=mcp-app",
    "text": "<!doctype html>..."
  }]
}
```

### Iframe 沙箱

宿主在一个沙箱化的 `<iframe>` 中渲染 HTML，采用：

- `sandbox="allow-scripts allow-same-origin"`（或按服务器声明更严格）
- 通过响应头应用服务器声明的 CSP。
- 没有 cookie，没有来自宿主源的 localStorage。
- 网络访问仅限于 CSP 中的 `connectSrc`。

### postMessage 协议

iframe 通过 `window.postMessage` 与宿主通信。一种小巧的 JSON-RPC 2.0 方言：

始终将 `targetOrigin` 固定为对端的确切源，并在接收端处理任何载荷之前，根据允许列表校验 `event.origin`。绝不要在此通道的任何一端使用 `"*"`——消息体携带的是工具调用和资源读取。

```js
// iframe to host  (pin to host origin)
window.parent.postMessage({
  jsonrpc: "2.0",
  id: 1,
  method: "host.callTool",
  params: { name: "notes_update", arguments: { id: "note-14", title: "..." } }
}, "https://host.example.com");

// host to iframe  (pin to iframe origin)
iframe.contentWindow.postMessage({
  jsonrpc: "2.0",
  id: 1,
  result: { content: [...] }
}, "https://iframe.example.com");

// receiver on both sides
window.addEventListener("message", (event) => {
  if (event.origin !== "https://expected-peer.example.com") return;
  // safe to process event.data
});
```

UI 可调用的宿主侧方法：

- `host.callTool(name, arguments)` — 调用服务器工具。
- `host.readResource(uri)` — 读取一个 MCP 资源。
- `host.getPrompt(name, arguments)` — 获取一个提示词模板。
- `host.close()` — 关闭 UI。

每次调用仍然经过 MCP 协议，并继承服务器的权限。

### 权限

`_meta.ui.permissions` 列表请求额外能力：

- `camera` — 访问用户的摄像头（用于扫描文档类 UI）。
- `microphone` — 语音输入。
- `geolocation` — 定位。
- `network:*` — 比单独的 `connectSrc` 允许的更广的网络访问。

每项权限都是用户在 UI 渲染前会看到的一个提示。

### 安全风险

iframe 中的 HTML 仍然是 HTML。新增的攻击面：

- **通过 UI 进行提示注入。** 恶意的服务器 UI 可以显示看起来像系统消息的文本来欺骗用户。宿主渲染应当在视觉上区分服务器 UI 与宿主 UI。
- **通过 `connectSrc` 外泄。** 如果 CSP 允许 `connect-src: *`，UI 就能把数据发往任何地方。默认应当严格。
- **点击劫持。** UI 覆盖在宿主界面之上。宿主必须阻止 z-index 操纵并强制执行不透明度规则。
- **窃取焦点。** UI 夺取键盘焦点并捕获下一条消息。宿主必须拦截。

Phase 13 · 15 将这些作为 MCP 安全的一部分深入讲解；本课只做引入。

### `ui/initialize` 握手

iframe 加载后，它通过 postMessage 发送 `ui/initialize`：

```json
{"jsonrpc": "2.0", "id": 0, "method": "ui/initialize",
 "params": {"theme": "dark", "locale": "en-US", "sessionId": "..."}}
```

宿主以能力和一个会话令牌响应。UI 在后续每次宿主调用中使用该会话令牌。

### AppRenderer / AppFrame SDK 原语

ext-apps SDK 暴露两个便利原语：

- `AppRenderer`（服务器端）— 包装一个 React / Vue / Solid 组件，并发射一个带有正确 MIME 和元数据的 `ui://` 资源。
- `AppFrame`（客户端）— 接收资源，挂载 iframe，并居中协调 postMessage。

你可以使用它们，也可以手写 HTML 和 JSON-RPC。

### 生态系统现状

MCP Apps 于 2026 年 1 月 26 日发布。截至 2026 年 4 月的客户端支持情况：

- **Claude Desktop。** 自 2026 年 1 月起完整支持。
- **ChatGPT。** 通过 Apps SDK 完整支持（底层为同一个 MCP Apps 协议）。
- **Cursor。** Beta；通过设置启用。
- **VS Code。** 仅 Insider 构建。
- **Goose。** 完整支持。
- **Zed、Windsurf。** 已列入路线图。

生产中的服务器：仪表盘、地图可视化、数据表格、图表构建器、沙箱 IDE 预览。

## 动手用

`code/main.py` 用一个 `visualize_timeline` 工具扩展了笔记服务器，该工具返回一个 `ui://notes/timeline` 资源，外加一个针对该 URI 的 `resources/read` 处理器，它返回一个小巧但完整的 HTML 包，内含一个 SVG 时间线。该 HTML 用标准库模板化——没有构建系统。postMessage 以 JS 注释勾勒，因为标准库无法驱动浏览器。

要关注的地方：

- 工具响应上的 `_meta.ui` 携带 resourceUri、CSP、permissions。
- HTML 在没有网络访问的情况下渲染；所有数据都内联。
- JS 通过 `window.parent.postMessage` 调用 `host.callTool`（有文档说明，但在这个标准库演示中是惰性的）。

## 交付物

本课产出 `outputs/skill-mcp-apps-spec.md`。给定一个能从交互式 UI 中获益的工具，该技能产出完整的 MCP Apps 契约：`ui://` URI、CSP、permissions、postMessage 入口点，以及一份安全检查清单。

## 练习

1. 运行 `code/main.py` 并检查发射出的 HTML。直接在浏览器中打开该 HTML；验证 SVG 能渲染。然后勾勒 UI 用来调用 `host.callTool("notes_update", ...)` 的 postMessage 契约。

2. 收紧 CSP：移除 `'unsafe-inline'`，改用基于 nonce 的脚本策略。HTML 生成代码中会有什么变化？

3. 添加第二个 UI 资源 `ui://notes/editor`，带有一个就地编辑笔记的表单。当用户提交时，iframe 调用 `host.callTool("notes_update", ...)`。

4. 审计 UI 的攻击面。恶意服务器可能在哪里注入内容？iframe 沙箱能防御什么，又防御不了什么？

5. 阅读 SEP-1724 规范，找出 MCP Apps SDK 中这个玩具实现没有用到的一项能力。（提示：组件级状态同步。）

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| MCP Apps | "交互式 UI 资源" | 于 2026-01-26 发布的 SEP-1724 扩展 |
| `ui://` | "应用 URI 方案" | 用于 UI 包的资源方案 |
| `text/html;profile=mcp-app` | "那个 MIME" | MCP App HTML 的内容类型 |
| Iframe 沙箱 | "渲染容器" | 浏览器对 UI 的沙箱化，带 CSP 和权限 |
| postMessage JSON-RPC | "UI 到宿主的线路" | 用于宿主调用的小巧 JSON-RPC-over-postMessage 方言 |
| `_meta.ui` | "工具-UI 绑定" | 将工具结果链接到 UI 资源的元数据 |
| CSP | "Content-Security-Policy" | 声明脚本、网络、样式的允许来源 |
| AppRenderer | "服务器 SDK 原语" | 将框架组件转换为 `ui://` 资源 |
| AppFrame | "客户端 SDK 原语" | 居中协调 postMessage 的 iframe 挂载助手 |
| `ui/initialize` | "握手" | UI 发往宿主的第一条 postMessage |

## 延伸阅读

- [MCP ext-apps — GitHub](https://github.com/modelcontextprotocol/ext-apps) — 参考实现与 SDK
- [MCP Apps specification 2026-01-26](https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx) — 正式规范文档
- [MCP — Apps extension overview](https://modelcontextprotocol.io/extensions/apps/overview) — 高层文档
- [MCP blog — MCP Apps launch](https://blog.modelcontextprotocol.io/posts/2026-01-26-mcp-apps/) — 2026 年 1 月发布博文
- [MCP Apps API reference](https://apps.extensions.modelcontextprotocol.io/api/) — JSDoc 风格的 SDK 参考
