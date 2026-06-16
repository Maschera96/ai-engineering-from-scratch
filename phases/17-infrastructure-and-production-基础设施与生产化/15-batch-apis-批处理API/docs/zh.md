# Batch APIs：50% 折扣成为行业标准

> Every major 提供商 ships an async 批处理 API with a 50% discount and ~24-小时 turnaround. OpenAI, Anthropic, Google, and most of the inference 平台 (Fireworks 批处理 tier, Together 批处理) implement the same 模式. Stack 批处理 搭配 提示词缓存 and overnight pipelines drop to ~10% of synchronous-uncached 成本. The rule is brutally 简单: if it is not interactive, it belongs on 批处理. Content generation pipelines, document classification, data extraction, report generation, bulk labeling, 目录 tagging ： anything tolerant of 24-小时 延迟 is money left on the table until it moves to 批处理. The 2026 生产化 模式 is to triage every new LLM 工作负载 into three lanes: interactive (synchronous with caching), semi-interactive (async 队列 with fallback), 批处理 (overnight, cached 输入 stacked). 工作负载 that pretend to be interactive but tolerate minutes of 延迟 waste most.

**Type:** Learn
**Languages:** Python（标准库， 玩具 batch-vs-sync cost模拟器)
**Prerequisites:** Phase 17 · 14 (Prompt & Semantic Caching)
**Time:** ~45 分钟

## 学习目标

- Name the three 提供商 批处理 APIs (OpenAI, Anthropic, Google) and the common 50% discount + 24h turnaround guarantees.
- Compute 这个成本 for stacking 批处理 + cached-输入 on an overnight classification 工作负载 和 比较 to synchronous-uncached baseline.
- Triage 一个工作负载 into interactive / semi-interactive / 批处理 和 说明理由 the lane.
- Name the two traps: partial interactivity (user expects faster than 24h) 和 输出-schema drift (批处理 file format differs per 提供商).

## 问题

Your 团队 ships a nightly report generation pipeline. 50,000 documents, summarize each, cluster the summaries, draft an executive brief. Running synchronously it takes 4 hours at $2,000/night. You hear about 批处理 APIs.

这个批处理 gets you 50% off. You also enable 提示词缓存 on the system 提示词 (shared across all 50k calls). Stacked, the bill drops to $180/night ： ~9% of baseline. Same pipeline, three config changes.

批处理 is the cheapest lever in the LLM 成本 toolkit that nobody pulls. The reason is mostly organizational: 团队 think "real-time" when the SLA actually is "by morning." This lesson is about not leaving 90% of the bill on the table.

## 概念

### 三种 Batch API

**OpenAI 批处理 API**: JSONL file upload with a list of 请求. Promised 24-小时 turnaround (通常 ~2-8 hours in practice). 50% discount on 输入 和 输出 词元. `/v1/batches` endpoint. Cache-eligible 输入 also get cached-输入 定价 on top.

**Anthropic Message Batches**: JSONL upload. 24-小时 turnaround. 50% discount. Supports `cache_control` ： cache writes are explicit, reads happen automatically within 这个批处理.

**Google Vertex AI 批处理 Prediction**: BigQuery or GCS 输入. 类似 50% discount for Gemini. Integrates with Vertex pipelines.

### 语义：异步，不是慢

批处理 is "I 承诺 to return within 24 hours" ： not "this will take 24 hours." Typical P50 is 2-6 hours. 提供商 schedules your 批处理 during off-peak windows when GPU inventory is underutilized.

### 与缓存叠加

A 50k-document summarization with the same 4K-词元 system 提示词:

- Synchronous uncached: 50000 × ($输入 × 4000 + $输出 × 200) at full rates.
- Synchronous cached: system 提示词 cached after first write; remaining 49999 get 10x 更便宜 输入.
- 批处理 cached: all of the above plus 50% discount on both read and write.

The stack: 批处理 + cache = ~10% of sync uncached bill. Any 工作负载 that runs overnight and has a shared system 提示词 should use this.

### 工作负载分诊

**Interactive** ： user waits for the response. TTFT matters. Synchronous call with 提示词缓存. Cannot 批处理.

**Semi-interactive** ： user submits a task, checks back in minutes. Async 队列 with fallback to sync if 批处理 not 可用. Think moderate-volume RAG indexing.

**批处理** ： user expects results "by morning" 或 "next 小时." Content pipelines, classification at scale, offline analysis. Always 批处理, always stack caching.

Common mistake: classifying everything as interactive because the pipeline is 生产化. 生产化 is not 一个延迟 spec ： SLA is.

### The partial-interactivity trap

Some 功能 look interactive but tolerate 5-10 minutes. Example: a nightly 客户 health report with "refresh" button. User clicks refresh; wait 10 minutes is fine. 团队 ships it as synchronous. 50 concurrent refreshes 成本 10x what batched-and-delivered-via-email would 成本.

The question to ask: "What does 24-小时 均值 for this user?" If the answer is "they wouldn't notice," 批处理 it.

### 这个输出-schema trap

批处理 file formats differ per 提供商:

- OpenAI: JSONL, one 请求 per line.
- Anthropic: JSONL, one message per line; response format embedded.
- Vertex: BigQuery table or GCS prefix with TFRecord.

Writing "one 批处理 client" 跨 提供商 means adapter code per 提供商. Gateways that advertise multi-提供商 批处理 (Portkey, LiteLLM some tiers) still thin-wrap the raw format.

### 你应该记住的数字

- 批处理 discount across 提供商: 50% flat on 输入 + 输出.
- Turnaround SLA: 24 hours guaranteed, 2-6 hours typical P50.
- Stacked 批处理 + cached 输入: ~10% of sync uncached 成本.
- 工作负载 triage rule: if 24h 延迟 acceptable, always 批处理.

## 使用它

`code/main.py` computes costs across sync, sync+cache, 批处理, 和 批处理+cache for a 50k-document 工作负载. Reports 节省 in $ and percent.

## 交付它

This lesson 产出 `outputs/skill-batch-triager.md`. 给定工作负载 characteristics, triages into interactive/semi/批处理 and estimates 节省.

## 练习

1. Run `code/main.py`. For a 100k-doc pipeline with 3K-词元 system 提示词 and 500-词元 输出, compute 这个节省 of full stack (批处理 + cache) 对比 sync baseline.
2. Pick three 功能 in a real 产品 you know. Triage each into interactive/semi/批处理.
3. A user complains their report took 3 hours. Was that 一个批处理 mis-triage or a legitimate interactive? Write 这个决策 criterion.
4. Your 批处理 API return SLA is 24h but P99 is 20 hours. How do you communicate this to the user ： what is the downstream system behavior on the edge case?
5. Compute 盈亏平衡: at what shared-prefix length does 批处理 + cache become 更便宜 than running overnight on your own 预留 GPU?

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| 批处理 API | "async discount" | 50% off with 24h turnaround |
| JSONL | "批处理 format" | One JSON 请求 per line; OpenAI/Anthropic standard |
| Message Batches | "Anthropic 批处理" | Anthropic's 批处理 API 产品 name |
| 批处理 prediction | "Vertex 批处理" | Vertex AI's 批处理 API 产品 |
| Turnaround SLA | "24h 承诺" | Guarantee, not typical; typical is 2-6h |
| 工作负载 triage | "interactivity 决策" | Interactive / semi / 批处理 路由 决策 |
| 输出 schema | "response format" | Per-提供商 JSONL layout; not portable |
| Stacked discount | "批处理 + cache" | ~10% of uncached sync bill when both apply |

## 延伸阅读

- [OpenAI Batch API](https://platform.openai.com/docs/guides/batch) ： JSONL format and `/v1/batches` semantics.
- [Anthropic Message Batches](https://docs.anthropic.com/en/docs/build-with-claude/batch-processing) ： 批处理 format and `cache_control` interaction.
- [Vertex AI Batch Prediction](https://cloud.google.com/vertex-ai/generative-ai/docs/model-reference/batch-prediction) ： Gemini 批处理 semantics.
- [Finout — OpenAI vs Anthropic API Pricing 2026](https://www.finout.io/blog/openai-vs-anthropic-api-pricing-comparison)
- [Zen Van Riel — LLM API Cost Comparison 2026](https://zenvanriel.com/ai-engineer-blog/llm-api-cost-comparison-2026/)
