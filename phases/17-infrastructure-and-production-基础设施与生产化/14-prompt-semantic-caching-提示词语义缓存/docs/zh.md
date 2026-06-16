# 提示词缓存与语义缓存经济学

> **定价 snapshot dated 2026-04.** Numeric claims below reflect vendor rate cards captured at this lesson's publication; verify against the linked docs before quoting them downstream.

> Caching happens at two layers. L2 (提供商-level) 提示词/prefix caching reuses attention KV for repeated prefixes ： Anthropic's 提示词-caching docs advertise up to 90% 成本 reduction and 85% 延迟 reduction on long 提示词; for Claude 3.5 Sonnet cache reads are $0.30/M 对比 $3.00/M fresh with a 5-minute TTL and a 2x write premium for the 1-小时 TTL option (docs.anthropic.com, 2026-04). OpenAI 提示词缓存 applies automatically for 提示词 ≥1024 词元 and prices cached 输入 at roughly a 90% discount 对比 fresh (平台.openai.com, 2026-04); the exact per-模型 cached rate depends on the live rate card. L1 (app-level) 语义缓存 skips the LLM entirely on embedding similarity hits. Vendor "95% accuracy" refers to match correctness, not hit rate ： reported 生产化 hit rates range from 10% (open-ended chat) up to 70% (structured FAQ); neither 提供商 publishes an official baseline, so treat these as community telemetry rather than guarantees. 这个生产化 pitfalls: parallelization kills caching (N parallel 请求 issued before the first cache write can inflate spend several-fold), and dynamic content inside the prefix prevents cache hits entirely. ProjectDiscovery reported moving from 7% to 74% hit rate (2025-11) by moving dynamic text out of the cacheable prefix.

**Type:** Learn
**Languages:** Python（标准库， 玩具 two-layer cache模拟器)
**Prerequisites:** Phase 17 · 04 (vLLM Serving Internals), Phase 17 · 06 (SGLang RadixAttention)
**Time:** ~60 分钟

## 学习目标

- Distinguish L2 提示词/prefix caching (KV reuse at 提供商) from L1 语义缓存 (LLM bypass on 类似 提示词).
- 解释 Anthropic's `cache_control` explicit marking and the two TTL options (5-min 对比 1-小时) with their price multipliers.
- Compute expected monthly 节省 given hit rate, 提示词/response mix, 和 词元 prices.
- Name the parallelization anti-模式 that inflates bills by 5-10x and the dynamic-content anti-模式 that collapses hit rate.

## 问题

You add 提示词缓存 to your RAG service. The bill stays flat. You measure the hit rate; it is 7%. Your 提示词 look static but they are not ： the system 提示词 includes the current date formatted to the minute, 一个请求 ID, and a randomized example reorder for diversity. Every 请求 writes a new cache entry, reads zero.

Separately, your agent runs ten parallel tool calls per user question. All ten arrive at 这个提供商 before the first cache write completes. Ten writes, zero reads. Your bill is 5-10x what "with caching" was supposed to 成本.

Caching is a protocol, not a flag. Two layers, two different failure modes.

## 概念

### L2：提供商提示词/前缀缓存

提供商 stores the attention KV for a cacheable prefix and reuses it on the next 请求 that matches the prefix. You pay a write 成本 once, reads nearly free.

**Anthropic (Claude 3.5 / 3.7 / 4 series)**: explicit `cache_control` marker in 这个请求. You tag which blocks are cacheable. TTL: 5-minute (write costs 1.25x base) or 1-小时 (write costs 2x base). Cache reads: $0.30/M on Claude 3.5 Sonnet 对比 $3.00/M fresh ： 10x 更便宜 (docs.anthropic.com, as of 2026-04). Rates differ per 模型 (Opus/Haiku published separately); always cross-check the live 定价 page.

**OpenAI**: automatic caching for 提示词 ≥1024 词元 (平台.openai.com, 2026-04). No explicit flag. Cached 输入 is roughly 10x 更便宜 than fresh on current gpt-4o/gpt-5 rate cards. Neither docs nor release notes publish an official hit-rate baseline; community reports cluster around 30–60% with careful 提示词 design. Monitor `usage.cached_tokens` to measure your own.

**Google (Gemini)**: context caching via explicit API; 1M-词元 context means caching pays even more.

**自托管 (vLLM, SGLang)**: Phase 17 · 06 covers RadixAttention ： same 模式 at your own compute.

### L1：应用级语义缓存

Before calling the LLM at all, hash 这个提示词, embed it, and look for 一个类似 cached 请求 (cosine similarity above threshold, typically 0.95+). On hit, return the cached response. On miss, call LLM and cache the result.

Open-source: Redis Vector Similarity, GPTCache, Qdrant. Commercial: Portkey Cache, Helicone Cache.

Vendor accuracy claims refer to how often the returned cached response was semantically 合适： not how often you hit. 生产化 hit rates:

- Open-ended chat: 10-15%.
- Structured FAQ / support: 40-70%.
- Code questions: 20-30% (small variants kill hits).
- Voice agents repeating 提示词: 50-80% (voice normalization fixed set).

### 并行化反模式

Your agent makes 10 tool calls in parallel. All 10 have the same 4K-词元 system 提示词. Anthropic cache writes are per-请求; the first cache-write completes around 300 ms after 这个提供商 sees 这个提示词. 请求 2-10 arrive in the same millisecond window and each sees cache miss. You pay 10 write premiums, 0 read discounts.

Fix: 批处理 with sequential-first ： make 请求 1 alone, then fire 2-10 once 1's cache has populated. Adds 300 ms to the first tool call; saves 5-10x the bill.

### 动态内容反模式

Your system 提示词 looks like:

```
You are a helpful assistant. The current time is 14:32:17.
User ID: abc123. Today is Tuesday...
```

Every 请求 is unique. Every 请求 writes. Zero hits.

Fix: move everything truly static to the cacheable prefix; append dynamic content after the cache boundary:

```
[cacheable]
You are a helpful assistant. [rules, examples, instructions]
[/cacheable]
[dynamic, not cached]
Current time: 14:32:17. User: abc123.
```

ProjectDiscovery moved from 7% to 74% cache hit rate this way and published the anatomy.

### 为夜间工作负载叠加批处理与缓存

批处理 APIs (Phase 17 · 15) give 50% discount at 24-小时 turnaround. Cached 输入 on top gets you ~10x on top of that. Overnight classification, labeling, and report generation 工作负载 can drop to ~10% of synchronous-uncached 成本 by stacking.

### 你应该记住的数字

定价 points are captured 2026-04 from the linked vendor docs and drift every few months ： re-check before relying on them.

- Anthropic cached read: $0.30/M on Claude 3.5 Sonnet, roughly 10x 更便宜 than fresh 输入 (docs.anthropic.com).
- Anthropic cache write premium: 1.25x (5-min TTL) or 2x (1-小时 TTL).
- OpenAI auto-cache: applies to 提示词 ≥1024 词元; cached 输入 priced at roughly 10% of fresh 输入 on current rate cards (平台.openai.com).
- Semantic cache hit rate (community-reported): ~10% open chat; up to ~70% structured FAQ. Not a vendor-documented baseline.
- ProjectDiscovery: 7% → 74% hit rate by moving dynamic out of prefix (project blog, 2025-11).
- Parallelization anti-模式: typical reports of 5–10x bill inflation when N parallel 请求 miss the first cache write.

## 使用它

`code/main.py` simulates L1 + L2 caching on mixed 工作负载. Reports hit rates, bill, and shows the parallelization penalty.

## 交付它

This lesson 产出 `outputs/skill-cache-auditor.md`. 给定提示词 template and 流量, audits cacheability and recommends restructure.

## 练习

1. Run `code/main.py`. Toggle the parallelization flag. How much does the bill change?
2. Your system 提示词 has a date. Move it out. Show before/after hit rate math.
3. Calculate 盈亏平衡 for 1-小时 TTL (2x write) 对比 5-minute TTL (1.25x write) given your 请求 arrival rate.
4. Semantic cache at 0.95 threshold hits 20%. At 0.85 it hits 50% but you see incorrect cached responses. Pick the right threshold and 说明理由.
5. You 批处理 10 parallel sub-queries per user question. Rewrite for cache-friendliness without adding end-to-end 延迟.

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| L2 提示词 cache | "前缀缓存" | 提供商 stores KV for repeated prefix |
| `cache_control` | "Anthropic cache marker" | Explicit attribute marking cacheable blocks |
| Cache write premium | "write tax" | Extra 成本 for first miss-to-cache (1.25x or 2x) |
| L1 semantic cache | "embedding cache" | App-level hash-and-embed before calling LLM |
| GPTCache | "LLM caching lib" | Popular OSS L1 cache library |
| Cache hit rate | "hits / total" | Fraction of 请求 served from cache |
| Parallelization anti-模式 | "the N-write trap" | N parallel 请求 miss cache N times |
| Dynamic content trap | "the time-in-提示词 trap" | Dynamic bytes in prefix kill hit rate |
| RadixAttention | "intra-replica cache" | SGLang's prefix-cache implementation |

## 延伸阅读

- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) ： official `cache_control` semantics and TTLs.
- [OpenAI Prompt Caching](https://platform.openai.com/docs/guides/prompt-caching) ： automatic caching behavior and eligibility.
- [TianPan — Semantic Caching for LLMs Production](https://tianpan.co/blog/2026-04-10-semantic-caching-llm-production)
- [ProjectDiscovery — Cut LLM Costs 59% With Prompt Caching](https://projectdiscovery.io/blog/how-we-cut-llm-cost-with-prompt-caching)
- [DigitalOcean / Anthropic — Prompt Caching](https://www.digitalocean.com/blog/prompt-caching-with-digital-ocean)
