---
name: a2a-agent-spec-zh
description: 为应当可通过 A2A 调用的智能体生成 Agent Card 与 skills 模式。
version: 1.0.0
phase: 13
lesson: 18
tags: [a2a, agent-card, task-lifecycle, delegation]
---

给定一个智能体的能力及其预期的协作方，生成它的 A2A Agent Card 与 skill 定义。

生成：

1. Agent Card。`name`、`description`、`url`、`version`、`schemaVersion`、`capabilities`（streaming、pushNotifications）、`skills[]`。
2. Skills 列表。每个包含 `id`、`name`、`description`、`inputModes`、`outputModes`。在描述中使用 "Use when X. Do not use for Y." 模式。
3. 任务状态计划。为每个 skill 给出预期的状态转换以及 input_required 路径。
4. 签名计划。是否通过 AP2 对 card 签名（对于可被外部调用的智能体，推荐签名）。
5. 传输。JSON-RPC over HTTP（默认）或 gRPC。注明与 v1.0 的向后兼容性。

硬性拒绝项：
- 任何没有稳定 URL 的 Agent Card。会破坏发现机制。
- 任何未声明输入与输出模式的 skill。调用方无法推断兼容性。
- 任何可被外部调用却没有 AP2 签名计划的智能体。存在冒充攻击面。

拒绝规则：
- 如果该智能体的用例只是单次工具调用，拒绝为其搭建 A2A；改为推荐 MCP。
- 如果该智能体暴露了它不应暴露的内部信息（工具调用轨迹、思维链），拒绝并强制其保持不透明。
- 如果该智能体需要用 A2A 进行支付（AP2 用例），确认 AP2 扩展版本，并指出 AP2 与核心 A2A 是相互独立的。

输出：一页式的 Agent Card JSON、每个操作的 skills 模式、状态转换计划、签名与传输选择。最后给出该智能体承诺的最低 v1.0 向后兼容性保证。
