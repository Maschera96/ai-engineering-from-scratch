---
name: mcp-transport-migrator-zh
description: 制定从传统 HTTP+SSE 迁移到 Streamable HTTP 的迁移方案，保证会话 id 的连续性并进行 Origin 校验。
version: 1.0.0
phase: 13
lesson: 09
tags: [mcp, streamable-http, sse-migration, session-id, origin]
---

给定一个现有的 HTTP+SSE（传统）MCP 服务器，制定迁移到单端点 Streamable HTTP 的方案。

产出：

1. 端点重写。将 `/messages` 和 `/sse` 合并为一个 `/mcp`。把 POST 映射到请求处理，GET 映射到 SSE 流，DELETE 映射到会话终止。
2. 会话连续性。在首次 POST 时生成新的 `Mcp-Session-Id`。拒绝客户端提供的 id。如果客户端先发送传统会话 cookie，则保留桥接逻辑。
3. Origin 校验。将明确的生产环境 origin 列入白名单（`https://app.company.com`、`https://claude.ai`、localhost 变体）。其他全部返回 403 拒绝。
4. Last-event-id 重放。为每个会话保留最近事件的环形缓冲区，使重连能够恢复。
5. 弃用窗口期。记录切换日期以及 60 天的宽限期，在此期间传统端点以 301 跳转到新端点并附带一个 warning 响应头。

硬性拒绝：
- 任何无限期保留两个端点的方案。传统 SSE 将在 2026 年移除。
- 任何由客户端生成会话 id 的方案。会破坏加密随机性要求。
- 任何没有 Origin 校验的方案。存在 DNS 重绑定漏洞。

拒绝规则：
- 如果服务器仅限本地（stdio），拒绝迁移到 HTTP；本地场景下 stdio 是正确选择。
- 如果服务器尚未提供 OAuth，请先完成 Phase 13 · 16 再公开暴露。
- 如果托管目标不支持长连接 HTTP（例如 Vercel 免费层），拒绝并推荐 Cloudflare Workers。

输出：一份迁移运行手册，包含端点变更、Origin 白名单、会话 id 方案、弃用时间表，以及一份测试清单，覆盖 initialize、tools/list、流式通知、带 last-event-id 的重连，以及显式 DELETE。
