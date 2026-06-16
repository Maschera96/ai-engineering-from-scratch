---
name: long-video-strategy-planner-zh
description: 为长视频理解任务在 brute-context、ring-attention、token-compression 或 agentic-retrieval 之间做出选择，并计算延迟与召回率预期。
version: 1.0.0
phase: 12
lesson: 18
tags: [long-video, gemini, ring-attention, videoagent, retrieval]
---

给定视频时长、查询复杂度（单一事件 vs 整体摘要）以及开放还是封闭的约束，选择一种长视频策略并产出一份配置。

产出：

1. 策略选型。Brute-context、ring-attention（LongVILA）、token-compression（Video-XL）或 agentic-retrieval（VideoAgent）。
2. 词元预算。Duration * FPS * per-frame-tokens。如果 > LLM 上下文则发出警告。
3. 预期召回率。在视频长度各百分位上的大海捞针（needle-in-a-haystack）召回率。相关时引用 Gemini 1.5 报告。
4. 延迟。brute-context 的预填充（prefill）时间；agentic 的检索 + VLM 时间。
5. 工程路径。为所选策略提供代码片段脚手架。
6. 回退方案。混合方案：brute-context 全局摘要 + agentic 局部细节。

硬性拒绝：
- 为一段 2 小时的视频在开放的 72B 模型上提议 brute-context。上下文放不下。
- 声称 agentic 检索总是更优。对于整体摘要类问题，它不如 brute context。
- 在未指出召回率代价的情况下推荐 token 压缩。

拒绝规则：
- 如果目标是一段 90 分钟视频且要求前沿级召回率（>95%），拒绝仅开放方案并推荐 Gemini 2.5 Pro。
- 如果用户无法负担工具调用循环，拒绝 agentic-retrieval 并提议压缩后的 brute-context。
- 如果用户需要实时（边播边处理），拒绝检索（太慢）并推荐流式 Qwen2.5-VL。

输出：一页式方案，包含策略、预算、召回率、延迟、工程路径和回退方案。结尾附上 arXiv 2403.05530（Gemini 1.5）和 2403.10517（VideoAgent）以供对比。
