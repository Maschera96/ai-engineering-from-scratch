---
name: mcp-apps-spec-zh
description: 为需要交互式 UI 资源的工具产出完整的 MCP Apps 契约。
version: 1.0.0
phase: 13
lesson: 14
tags: [mcp, apps, ui-resources, csp, iframe-sandbox]
---

给定一个能从交互式 UI（时间线、表单、仪表盘、地图、图表）中受益的工具，产出 MCP Apps 契约。

产出：

1. `ui://` URI。为 UI 资源指定一个规范名称（例如 `ui://notes/timeline`）。
2. 工具结果形态。`content[]` 包含 `text` 前言和 `ui_resource` 块；填充 `_meta.ui`。
3. CSP。为 `default-src`、`script-src`、`connect-src`、`img-src`、`style-src` 列出最小允许列表。除非必要，否则避免 `'unsafe-inline'`。
4. 权限列表。如果需要，包含摄像头 / 麦克风 / 地理位置 / 网络；不需要则留空。
5. postMessage 入口点。UI 将发起哪些 `host.*` 调用以及它们返回什么。
6. 安全检查清单。与宿主区分、防点击劫持、严格的 connect-src、若渲染任何用户内容则进行 HTML 净化。

强制拒绝项：
- 使用 `default-src *` 的 CSP。安全风险敞口过大。
- 超出 UI 实际使用范围的任何 `permissions` 请求。最小权限原则。
- 任何加载外部脚本的 ui:// 资源。打包内联，否则拒绝。
- 任何在不净化的情况下渲染用户可控 HTML 的 UI。XSS 攻击向量。

拒绝规则：
- 如果该 UI 只是静态结果，拒绝搭建 App；返回文本内容。
- 如果该工具能从原生宿主控件（进度条、确认对话框）中受益，改为推荐这些控件。
- 如果宿主尚不支持 MCP Apps（截至 2026-04 的 VS Code stable、Zed、Windsurf），标注回退到文本的路径。

输出：一页契约，包含 `ui://` URI、工具结果 JSON、CSP、权限、postMessage 入口点和安全检查清单。以一句话说明能渲染此 UI 的最低宿主要求作为结尾。
