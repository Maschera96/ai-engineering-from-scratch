# 提示词 缓存 and 上下文 缓存

> 你的系统 提示词 is 4,000 词元. Your RAG 上下文 is 20,000 词元. You send both with every request. You also pay for both — every time. 提示词 缓存 lets the provider keep that prefix warm on their side and bill you 10% of the normal 速率 on reuse. Used correctly, it cuts 推理 成本 by 50–90% and first-token 延迟 by 40–85%.

**类型：** Build
**语言：** Python
**先修：** Phase 11 · 01 (提示词工程), Phase 11 · 05 (上下文工程), Phase 11 · 11 (缓存 and 成本)
**时间：** 约 60 分钟

## 问题

一个coding 智能体 sends the same 15,000-词元 系统 提示词 to Claude on every turn of a conversation. Twenty turns at $3/M 输入 词元 is $0.90 in 输入 成本 alone — before any of the 用户's actual 消息. Multiply by 10,000 daily conversations and the bill hits $9,000/day for 文本 that never changes.

你cannot shrink the 提示词 without hurting 质量. You cannot avoid sending it — the 模型 needs it on every turn. The only move is to stop paying full price for a prefix the provider has already seen.

那move is 提示词 缓存. Anthropic shipped it in August 2024 (with a 1-hour extended-TTL variant in 2025), OpenAI automated it later that year, Google shipped explicit 上下文 缓存 alongside Gemini 1.5, and all three now offer it as a first-class 特征 on their frontier 模型.

## 概念

![Prompt caching: write once, read cheap](../assets/prompt-caching.svg)

**The mechanic.** When a request's prefix matches one from a recent request, the provider serves the KV-cache from the previous run instead of re-encoding the 词元. You pay a small write premium the first time and a large read discount every time after.

**Three provider flavors in 2026.**

|Provider|API 风格|Hit discount|Write premium|Default TTL|Min cacheable|
|---------|-----------|--------------|---------------|-------------|---------------|
|Anthropic|Explicit `cache_control` markers on content 块|90% off 输入|25% surcharge|5 min (extendable to 1 hour)|1,024 词元 (Sonnet/Opus), 2,048 (Haiku)|
|OpenAI|Automatic prefix detection|50% off 输入|none|Up to 1 hour (best-effort)|1,024 词元|
|Google (Gemini)|Explicit `CachedContent` API|Storage-billed; read at ~25% of normal|Storage fee per 词元·hour|User-set (default 1 hour)|4,096 词元 (Flash), 32,768 (Pro)|

**The invariant.** All three 缓存 prefixes only. If any 词元 differs between requests, everything after the first differing 词元 is a miss. Put the *stable* parts at the top, the *variable* parts at the bottom.

### The cache-friendly layout

```text
[system prompt]          <-- cache this
[tool definitions]       <-- cache this
[few-shot examples]      <-- cache this
[retrieved documents]    <-- cache if reused, else don't
[conversation history]   <-- cache up to last turn
[current user message]   <-- never cache (different every time)
```

Violate the order — put the 用户 消息 above the 系统 提示词, interleave dynamic retrievals between few-shots — and the 缓存 never hits.

### The break-even calculation

Anthropic's 25% write premium means a cached 块 has to be read at least twice to net-save money. 1 write + 1 read averages 0.675x 成本 per request (saves 32%); 1 write + 10 reads averages 0.205x (saves 80%). Rule of thumb: 缓存 anything you expect to reuse at least 3 times within the TTL.

## 动手构建

### 步骤 1: Anthropic 提示词 缓存 with explicit markers

```python
import anthropic

client = anthropic.Anthropic()

SYSTEM = [
    {
        "type": "text",
        "text": "You are a senior Python reviewer. Follow the rubric exactly.\n\n" + RUBRIC_15K_TOKENS,
        "cache_control": {"type": "ephemeral"},
    }
]

def review(code: str):
    return client.messages.create(
        model="claude-opus-4-7",
        max_tokens=1024,
        system=SYSTEM,
        messages=[{"role": "user", "content": code}],
    )
```

这个`cache_control` marker tells Anthropic to store the 块 for 5 分钟. Reuse within that window hits; reuse after expires and writes again.

**响应 usage fields:**

```python
response = review(code_a)
response.usage
# InputTokensUsage(
#     input_tokens=120,
#     cache_creation_input_tokens=15023,   # paid at 1.25x
#     cache_read_input_tokens=0,
#     output_tokens=340,
# )

response_b = review(code_b)
response_b.usage
# cache_creation_input_tokens=0
# cache_read_input_tokens=15023           # paid at 0.1x
```

Check both fields in CI — if `cache_read_input_tokens` stays at zero across requests, your 缓存 keys are drifting.

### 步骤 2: one-hour extended TTL

For long-running 批次 jobs, the 5-分钟 default expires between jobs. Set `ttl`:

```python
{"type": "text", "text": RUBRIC, "cache_control": {"type": "ephemeral", "ttl": "1h"}}
```

1-hour TTL 成本 2x the write premium (50% over 基线 instead of 25%) but pays back fast on any 批次 reusing the prefix more than 5 times.

### 步骤 3: OpenAI automatic 缓存

OpenAI gives you nothing to configure. Any prefix over 1,024 词元 that matches a recent request gets a 50% discount automatically.

```python
from openai import OpenAI
client = OpenAI()

resp = client.chat.completions.create(
    model="gpt-5",
    messages=[
        {"role": "system", "content": SYSTEM_PROMPT},   # long and stable
        {"role": "user", "content": user_msg},
    ],
)
resp.usage.prompt_tokens_details.cached_tokens  # the discounted portion
```

Same cache-friendly layout rule applies. Two things kill OpenAI's 缓存 that don't kill Anthropic's: changing the `user` field (used as a 缓存 key component) and reordering 工具.

### 步骤 4: Gemini explicit 上下文 缓存

Gemini treats the 缓存 as a first-class object you create and name:

```python
from google import genai
from google.genai import types

client = genai.Client()

cache = client.caches.create(
    model="gemini-3-pro",
    config=types.CreateCachedContentConfig(
        display_name="rubric-v3",
        system_instruction=RUBRIC,
        contents=[FEW_SHOT_EXAMPLES],
        ttl="3600s",
    ),
)

resp = client.models.generate_content(
    model="gemini-3-pro",
    contents=["Review this code:\n" + code],
    config=types.GenerateContentConfig(cached_content=cache.name),
)
```

Gemini charges storage per 词元·hour for as long as the 缓存 lives, and reads at ~25% of normal 输入 速率. This is the right shape when you reuse the same giant 提示词 across many sessions over days.

### 步骤 5: measuring hit 速率 in 生产

See `code/main.py` for a simulated three-provider accountant that tracks write/read/miss counts and computes blended 成本 per 1K requests. Gate deploys on a 目标 hit 速率 — most 生产 Anthropic setups should see >80% read fraction after 预热.

## Pitfalls that still ship in 2026

- **Dynamic timestamps at the top.** `"Current time: 2026-04-22 15:30:02"` at the top of the 系统 提示词. Every request misses. Move timestamps below the 缓存 breakpoint.
- **工具 reordering.** Serialize 工具 in a stable order — a dict reshuffle between deploys breaks every hit.
- **Free-text near-duplicates.** "You are helpful." vs "You are a helpful 助手." — one byte difference = full miss.
- **Too-small 块.** Anthropic enforces a 1,024-词元 floor (2,048 for Haiku). Smaller 块 silently do not 缓存.
- **Blind 成本 dashboards.** Split "输入 词元" into cached vs uncached. Otherwise a traffic drop looks like a 缓存 win.

## 实际使用

这个2026 缓存 stack:

|Situation|Pick|
|-----------|------|
|智能体 with stable 10k+ 系统 提示词, many turns|Anthropic `cache_control` with 5-min TTL|
|批次 job reusing a prefix for 30+ 分钟|Anthropic with `ttl: "1h"`|
|Serverless endpoints on GPT-5, no custom infra|OpenAI automatic (just make your prefix stable and long)|
|Multi-day reuse of a giant code/doc 语料库|Gemini explicit `CachedContent`|
|Cross-provider 备选方案|Keep the cacheable prefix layout identical across providers so any hit works|

Combine with 语义 缓存 (Phase 11 · 11) for the user-message 层: 提示词 缓存 handles *token-identical* reuse, 语义 缓存 handles *meaning-identical* reuse.

## 交付成果

Save `outputs/skill-prompt-caching-planner.md`:

```markdown
---
name: prompt-caching-planner
description: Design a cache-friendly prompt layout and pick the right provider caching mode.
version: 1.0.0
phase: 11
lesson: 15
tags: [llm-engineering, caching, cost]
---

Given a prompt (system + tools + few-shot + retrieval + history + user) and a usage profile (requests per hour, TTL needed, provider), output:

1. Layout. Reordered sections with a single cache breakpoint marked; explain which sections are stable, which are volatile.
2. Provider mode. Anthropic cache_control, OpenAI automatic, or Gemini CachedContent. Justify from TTL and reuse pattern.
3. Break-even. Expected reads per write within TTL; net cost vs no-cache with math.
4. Verification plan. CI assertion that cache_read_input_tokens > 0 on the second identical request; dashboard split by cached vs uncached tokens.
5. Failure modes. List the three most likely reasons the cache will miss in this setup (dynamic timestamp, tool reorder, near-duplicate text) and how you will prevent each.

Refuse to ship a cache plan that places a dynamic field above the breakpoint. Refuse to enable 1h TTL without a reuse count that makes the 2x write premium pay back.
```

## 练习

1. **Easy.** Take a 10-turn conversation with a 5,000-词元 系统 提示词 against Claude. Run it without `cache_control` and then with. Report the input-token bill for each.
2. **Medium.** Write a test harness that, given a 提示词 template and a request log, computes the expected hit 速率 and dollar savings per provider (Anthropic 5m, Anthropic 1h, OpenAI automatic, Gemini explicit).
3. **Hard.** Build a layout 优化器: given a 提示词 and a list of fields marked `stable=True/False`, rewrite the 提示词 to put a single 缓存 breakpoint at the maximum cache-friendly position without losing information. Verify on a 真实 Anthropic endpoint.

## Key Terms

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|提示词 缓存|"Makes long prompts cheap"|Reusing a provider-side KV-cache for 匹配 prefixes; 50-90% discount on repeated 输入 词元.|
|`cache_control`|"The Anthropic marker"|Content-block attribute that declares "everything up to here is cacheable"; `{"type": "ephemeral"}`.|
|缓存 write|"Paying the premium"|The first request that populates the 缓存; billed at ~1.25x 输入 速率 on Anthropic, free on OpenAI.|
|缓存 read|"The discount"|Subsequent requests 匹配 the prefix; billed at 10% (Anthropic), 50% (OpenAI), ~25% (Gemini).|
|TTL|"How long it lives"|Seconds the 缓存 stays warm; Anthropic 5m default (extendable 1h), OpenAI best-effort up to 1h, Gemini user-set.|
|Extended TTL|"1-hour Anthropic 缓存"|`{"type": "ephemeral", "ttl": "1h"}`; 2x write premium but worth it for 批次 reuse.|
|Prefix match|"Why my 缓存 missed"|Caches only hit when every 词元 from the start up to the breakpoint is byte-identical.|
|上下文 缓存 (Gemini)|"The explicit one"|Google's named, storage-billed 缓存 object; best for multi-day reuse of large corpora.|

## 延伸阅读

- [Anthropic — Prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) — `cache_control`, 1h TTL, break-even tables.
- [OpenAI — Prompt caching](https://platform.openai.com/docs/guides/prompt-caching) — automatic prefix 匹配.
- [Google — Context caching](https://ai.google.dev/gemini-api/docs/caching) — `CachedContent` API and storage pricing.
- [Anthropic engineering — Prompt caching for long-context workloads](https://www.anthropic.com/news/prompt-caching) — original launch post with 延迟 numbers.
- Phase 11 · 05 (上下文工程) — where to slice the 提示词 so the 缓存 can land.
- Phase 11 · 11 (缓存 and 成本) — pair 提示词 缓存 with a 语义 缓存 on 用户 消息.
- [Pope et al., "Efficiently Scaling Transformer Inference" (2022)](https://arxiv.org/abs/2211.05102) — the KV-cache 内存 模型 that 提示词 缓存 exposes to users; explains why a cached prefix is ~10× cheaper to reread than to recompute.
- [Agrawal et al., "SARATHI: Efficient LLM Inference by Piggybacking Decodes with Chunked Prefills" (2023)](https://arxiv.org/abs/2308.16369) — prefill is the phase 提示词 缓存 shortcuts; this paper explains why TTFT drops dramatically on 缓存 hit while TPOT is unaffected.
- [Leviathan et al., "Fast Inference from Transformers via Speculative Decoding" (2023)](https://arxiv.org/abs/2211.17192) — 提示词 缓存 sits alongside 推测解码, Flash 注意力, and MQA/GQA as levers that bend the 推理 成本 曲线; read this for the other three.
