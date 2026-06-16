---
name: prompt-caching-planner-zh
description: Design a cache-friendly 提示词 layout and pick the right provider 缓存 mode.
version: 1.0.0
phase: 11
lesson: 15
tags: [llm-engineering, caching, cost]
---

给定a 提示词 (系统 + 工具 + 少样本 + 检索 + history + 用户) and a usage profile (requests per hour, TTL needed, provider), 输出：

1. Layout. Reordered sections with a single 缓存 breakpoint marked; explain which sections are stable, which are volatile.
2. Provider mode. Anthropic cache_control, OpenAI automatic, or Gemini CachedContent. Justify from TTL and reuse pattern.
3. Break-even. Expected reads per write within TTL; net 成本 vs no-cache with math.
4. Verification plan. CI assertion that cache_read_input_tokens > 0 on the second identical request; dashboard split by cached vs uncached 词元.
5. 失败模式s. List the three most likely reasons the 缓存 will miss in this setup (dynamic timestamp, 工具 reorder, near-duplicate 文本) and how you will prevent each.

拒绝 ship a 缓存 plan that places a dynamic field above the breakpoint. 拒绝 enable 1h TTL without a reuse count that makes the 2x write premium pay back.
