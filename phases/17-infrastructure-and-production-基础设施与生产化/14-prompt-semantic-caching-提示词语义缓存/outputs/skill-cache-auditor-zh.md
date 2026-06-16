---
name: cache-auditor-zh
description: Audit an LLM 提示词 template and 流量 模式 for cacheability. Recommend 提示词 restructure, TTL choice, parallelization fix, and semantic-cache threshold.
version: 1.0.0
phase: 17
lesson: 14
tags: [caching, prompt-cache, semantic-cache, anthropic, openai, parallelization, ttl]
---

给定一个提示词 template, 流量 模式 (arrival rate, parallel factor), 和 提供商 (Anthropic, OpenAI, Gemini, 自托管 vLLM), produce a cache audit.

产出：

1. Prefix structure. Split the template into static (cacheable) and dynamic (non-cacheable) sections. Flag any dynamic content currently in the prefix and propose the rewrite.
2. TTL choice. Anthropic 5-min (1.25x write) 对比 1-小时 (2x write). Pick based on arrival rate ： 1-小时 wins when the prefix is reused within the 小时 consistently.
3. Parallelization audit. Count parallel 请求 with shared prefix. If N > 2 and parallel, require serialize-first-then-fanout 模式. Quantify the expected bill reduction.
4. Semantic cache choice. Decide if L1 is worth it. Open-ended chat: maybe not (low hit). Structured FAQ / support: yes. Set cosine threshold, start 0.95; tune downward only with response-质量 评估.
5. Expected 节省. Compute 每月$ delta 对比 no-cache baseline given current 流量 and projected hit rates.
6. Observable. One dashboard 指标 that catches regressions: L2 cache hit rate over last rolling 小时; alert if drops >20%.

硬性拒绝：
- Claiming "50% 节省" without computing expected hit rate and write premium. 拒绝 ： calculate per-layer.
- Leaving dynamic content in prefix when 一个简单 rewrite moves it out. 拒绝 to sign off.
- Firing parallel 请求 with shared prefix without serialize-first 模式. 拒绝 ： state the 5-10x bill inflation.

拒绝规则：
- If 这个提示词 is >80% dynamic content by 词元, 拒绝 to 承诺 cache 节省. Recommend 语义缓存 at best.
- If semantic cache threshold is dropped below 0.85 without response-质量 eval, 拒绝 ： hallucination cache 风险.
- If 这个提供商 does not support explicit cache_control (non-Anthropic, non-Gemini-v1) and auto-caching only, note that hit rate is opportunistic, not guaranteed.

输出： a one-page audit listing prefix rewrite, TTL, parallelization 模式, L1 threshold, expected 节省, observable. End with a quarterly review 建议: re-audit 提示词 after any template change.
