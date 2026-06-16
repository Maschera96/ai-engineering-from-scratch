---
name: parallel-call-safety-check-zh
description: 审计工具注册表的并行安全性。为每个工具标记 parallel_safe，记录顺序依赖，并标出下游限流风险。
version: 1.0.0
phase: 13
lesson: 03
tags: [parallel-tool-calls, streaming, correlation, rate-limits]
---

给定一个工具注册表（包含名称、描述和执行器的工具列表），返回一份带注释的副本，新增 `parallel_safe: bool`、`ordering_deps: [tool_name]` 和 `rate_limit_group: name` 字段。

产出：

1. 逐工具分类。针对每个工具，判定：在同一轮中可安全并行运行（纯读取、不同资源）；不安全（变更操作、共享资源、外部限流）。
2. 依赖图。识别某个工具的输出应当作为另一个工具输入的配对。同一轮内无法并行化。用 `ordering_deps` 标记。
3. 限流分组。命中同一下游 API 的工具共享一个分组。宿主应按分组而非按工具来限制并发上限。
4. 安全建议。针对每个不安全的工具，说明该轮应禁用并行、排队，还是按资源分片。
5. 厂商特定标志。当工具集合中存在任何不安全工具时，建议在 OpenAI 上设置 `parallel_tool_calls=false`，或在 Anthropic 上设置 `disable_parallel_tool_use=true`。

硬性拒绝：
- 任何在审计后没有分类的注册表。默认拒绝；未知即视为不安全。
- 任何在共享资源上的写路径工具被标记为 `parallel_safe: true`。会出现竞态条件。
- 任何命中限流外部 API 但没有 `rate_limit_group` 的工具。

拒绝规则：
- 如果被要求不经检查就把所有工具标记为并行安全，拒绝。
- 如果注册表中包含作用于同一资源的有后果的工具（同一路径上的 `delete_file` 和 `write_file`），拒绝并行化，并引导至 Phase 14 · 09 进行沙箱级别的串行化。
- 如果用户辩称他们的工具从不发生竞态，拒绝并要求其提供证明（测试、日志或形式化论证）。竞态会在生产环境中无声地发生。

输出：一份修订后的注册表，以 JSON blob 形式给出，每个工具带有三个新增字段，随后是一段简短总结，点明风险最高的并行化选择及推荐的缓解措施。最后给出针对当前轮的建议 `tool_choice` 覆盖值。
