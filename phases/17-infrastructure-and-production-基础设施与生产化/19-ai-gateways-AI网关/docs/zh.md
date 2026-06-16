# AI 网关：LiteLLM、Portkey、Kong AI Gateway、Bifrost

> 一个网关 sits between your apps and 模型 提供商. Core 功能 are 提供商 路由, fallback, retries, rate limiting, 密钥 references, 可观测性, 护栏. Market split in 2026: **LiteLLM** is MIT OSS with 100+ 提供商, OpenAI-compatible, but breaks down around ~2000 RPS (8 GB 内存, cascading failures in published 基准); best for Python, <500 RPS, dev/prototyping. **Portkey** is control-plane-positioned (护栏, PII redaction, jailbreak detection, audit trails), went Apache 2.0 open-source March 2026, 20-40 ms 延迟 overhead, $49/mo 生产化 tier. **Kong AI 网关** built on Kong 网关 ： Kong's own 基准 on same 12 CPUs: 228% faster than Portkey, 859% faster than LiteLLM; $100/模型/月 定价 (max 5 on Plus tier); 企业级-fit if you're already on Kong. **Bifrost** (Maxim AI) ： automatic retries with configurable backoff, fallback to Anthropic on OpenAI 429. **Cloudflare / Vercel AI Gateways** ： managed, zero-ops, basic retry. 数据驻留 drives the self-host 决策; Portkey and Kong sit in the middle with OSS + optional managed.

**Type:** Learn
**Languages:** Python（标准库， 玩具 gateway-routing模拟器)
**Prerequisites:** Phase 17 · 01 (Managed LLM Platforms), Phase 17 · 16 (Model Routing)
**Time:** ~60 分钟

## 学习目标

- Enumerate the six core 网关 功能 (路由, fallback, retries, 限流, 密钥, 可观测性, 护栏).
- Map four 2026 gateways (LiteLLM, Portkey, Kong AI, Bifrost) to scale ceilings and use cases.
- Cite the Kong 基准 (228% 对比 Portkey, 859% 对比 LiteLLM) 和 解释 why it matters for >500 RPS.
- Choose 自托管 对比 managed 给定数据驻留 and ops budget.

## 问题

Your 产品 calls OpenAI, Anthropic, and 一个自托管 Llama. Each 提供商 has a different SDK, error 模型, rate limit, and auth scheme. You want 故障切换 (if OpenAI 429s, try Anthropic), a single credential store, 统一 可观测性, 和 限流 per tenant.

Reinventing this at the app layer couples every service to every 提供商. 一个网关 layer consolidates it into one process with one API (typically OpenAI-compatible) that fans out to 提供商.

## 概念

### 六个核心功能

1. **提供商 路由** ： OpenAI, Anthropic, Gemini, 自托管, etc. behind one API.
2. **Fallback** ： on 429, 5xx, 或 质量 failure, retry elsewhere.
3. **Retries** ： exponential backoff, bounded attempts.
4. **限流** ： per-tenant, per-key, per-模型.
5. **密钥 references** ： pull credentials from vault at runtime (never in app).
6. **可观测性** ： OTel + GenAI attributes (Phase 17 · 13) + 成本归因.
7. **护栏** ： PII redaction, jailbreak detection, allowed-topics filters.

### LiteLLM ： MIT OSS, Python

- 100+ 提供商, OpenAI-compatible, 路由器 config, fallback, 基本可观测性.
- Breaks down around 2000 RPS in Kong's 基准; 8 GB 内存 footprint, cascading failures under sustained load.
- Best fit: Python app, <500 RPS, dev/staging gateways, experimental 路由.
- 成本: $0 for OSS; cloud free tier exists.

### Portkey ： control plane positioning

- Apache 2.0 OSS as of March 2026. 护栏, PII redaction, jailbreak detection, audit trails.
- 20-40 ms per-请求 延迟 overhead.
- $49/mo for 生产化 tier with 保留+ SLA.
- Best fit: 受监管行业 needing 护栏 + 可观测性 bundled.

### Kong AI 网关 ： the scale play

- Built on Kong 网关 (mature API 网关 产品, lua+OpenResty).
- Kong's own 基准 on 12-CPU equivalent: 228% faster than Portkey, 859% faster than LiteLLM.
- 定价: $100/模型/月, max 5 on Plus tier.
- Best fit: already on Kong; >1000 RPS; willing to license.

### Bifrost (Maxim AI)

- Automatic retries with configurable backoff.
- Fallback to Anthropic on OpenAI 429 is a canonical recipe.
- Newer entrant; commercial.

### Cloudflare AI 网关 / Vercel AI 网关

- Managed, zero-ops. Basic retry and 可观测性.
- Best fit: Edge-serving JavaScript apps on Cloudflare/Vercel.
- Limited compared to Kong/Portkey on 护栏 和 限流.

### 自托管与托管

数据驻留 is the forcing function. Healthcare and finance 默认 self-host (LiteLLM or Portkey OSS or Kong). Consumer products 默认 managed (Cloudflare AI 网关) or middle-tier (Portkey managed). Hybrid: 自托管 for regulated tenant, managed for others.

### 延迟预算

- LiteLLM: 5-15 ms overhead typical.
- Portkey: 20-40 ms overhead.
- Kong: 3-8 ms overhead.
- Cloudflare/Vercel: 1-3 ms overhead (edge advantage).

网关 延迟 直接 adds to TTFT. For TTFT P99 < 100 ms SLA, Kong or Cloudflare. For P99 < 500 ms, any.

### 限流语义很重要

简单 词元-bucket works up to moderate scale. 多租户 需要 sliding-window + burst allowance + per-tenant tiering. LiteLLM ships 词元-bucket; Kong ships sliding-window; Portkey ships tiered.

### 网关 + 可观测性 + 路由 compose

Phase 17 · 13 (可观测性) + 16 (模型 路由) + 19 (gateways) are the same layer in 生产化. Pick one tool that covers all three or wire them carefully: most 2026 deployments combine Helicone (可观测性) or Portkey (护栏) with Kong (scale) for split roles.

### 你应该记住的数字

- LiteLLM: breaks at ~2000 RPS, 8 GB 内存.
- Portkey: 20-40 ms overhead; Apache 2.0 since March 2026.
- Kong: 228% faster than Portkey, 859% faster than LiteLLM.
- Kong 定价: $100/模型/月, 5 max on Plus tier.
- Cloudflare/Vercel: 1-3 ms overhead at the edge.

## 使用它

`code/main.py` simulates 网关 路由 with fallback across 3 提供商 under 429/5xx injection. Reports 延迟, retry rate, and fallback hit rate.

## 交付它

This lesson 产出 `outputs/skill-gateway-picker.md`. Given scale, ops posture, 合规, 延迟 budget, picks 一个网关.

## 练习

1. Run `code/main.py`. Configure fallback from OpenAI→Anthropic→自托管. What's the expected hit rate at 5% 提供商 error rate?
2. Your SLA is TTFT P99 < 200 ms on a 300 ms baseline. Which gateways stay within budget?
3. A healthcare 客户 需要 自托管 + PII redaction + audit. Pick Portkey OSS or Kong.
4. 比较 LiteLLM 对比 Kong: at what RPS ceiling should 一个团队 migrate?
5. Design 一个限流 策略 for 一个多租户 SaaS: free tier, trial tier, paid tier. 词元-bucket or sliding-window?

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| 网关 | "API broker" | Process sitting between apps and 提供商 |
| LiteLLM | "the MIT one" | Python OSS, 100+ 提供商, breaks at 2K RPS |
| Portkey | "护栏 网关" | Control plane + 可观测性, Apache 2.0 |
| Kong AI 网关 | "the scale one" | Built on Kong 网关, 基准 leader |
| Bifrost | "Maxim's 网关" | Retries + Anthropic fallback recipe |
| Cloudflare AI 网关 | "edge managed" | Edge-deployed managed 网关, zero-ops |
| PII redaction | "data scrub" | Regex + NER mask before sending to 模型 |
| Jailbreak detection | "提示词 injection guard" | Classifier on user 输入 |
| Audit trail | "regulated log" | Immutable record of every LLM call |
| 词元-bucket | "简单 rate limit" | Refill-based rate limiter |
| Sliding-window | "precise rate limit" | Time-windowed rate limiter; better fairness |

## 延伸阅读

- [Kong AI Gateway Benchmark](https://konghq.com/blog/engineering/ai-gateway-benchmark-kong-ai-gateway-portkey-litellm)
- [TrueFoundry — AI Gateways 2026 Comparison](https://www.truefoundry.com/blog/a-definitive-guide-to-ai-gateways-in-2026-competitive-landscape-comparison)
- [Techsy — Top LLM Gateway Tools 2026](https://techsy.io/en/blog/best-llm-gateway-tools)
- [LiteLLM GitHub](https://github.com/BerriAI/litellm)
- [Portkey GitHub](https://github.com/Portkey-AI/gateway)
- [Kong AI Gateway docs](https://docs.konghq.com/gateway/latest/ai-gateway/)
