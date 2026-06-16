# 推理平台经济学：Fireworks、Together、Baseten、Modal、Replicate、Anyscale

> The 2026 inference market is no longer GPU time rental. It bifurcates into custom silicon (Groq, Cerebras, SambaNova), GPU 平台 (Baseten, Together, Fireworks, Modal), and API-first marketplaces (Replicate, DeepInfra). Fireworks raised price $1/hr per GPU on May 1, 2026, 和 $4B valuation on 10T+ 词元/day tells you the volume-driven 模型 works. Baseten closed $300M Series E at $5B in January 2026. The competitive positioning rule is 简单: Fireworks optimizes 延迟, Together optimizes 目录 breadth, Baseten optimizes 企业级 polish, Modal optimizes Python-native DX, Replicate optimizes 多模态 reach, Anyscale optimizes distributed Python. This lesson gives you a matrix you can hand a founder.

**Type:** Learn
**Languages:** Python（标准库， 玩具 per-call economics比较器)
**Prerequisites:** Phase 17 · 01 (Managed LLM Platforms), Phase 17 · 04 (vLLM Serving Internals)
**Time:** ~60 分钟

## 学习目标

- Name the three market segments (custom silicon, GPU 平台, API-first) and map each vendor to a segment.
- 解释 why 这个"per-词元" API 定价 模型 compresses toward the serving engine's 成本 curve, not the hardware's.
- Compute effective 成本 per 请求 across at least three vendors and 解释 when per-minute (Baseten, Modal) beats per-词元.
- Identify which 平台 is the right 默认 for 一个给定工作负载 (Serverless bursty, steady high-吞吐量, fine-tuned variants, 多模态).

## 问题

You evaluated managed hyperscaler 平台. You decided you need a narrower, faster 提供商 ： Fireworks for 延迟, Together for breadth, Baseten for a fine-tuned custom 模型. Now you have six real choices and 这个定价 pages do not line up. Fireworks shows $/M 词元; Baseten shows $/minute; Modal shows $/second; Replicate shows $/prediction. You cannot 比较 them head-to-head without modeling 这个工作负载.

Worse, the business 模型 behind each 定价 page is different. Fireworks runs its own custom engine (FireAttention) on shared GPUs; the per-词元 rate reflects their utilization curve. Baseten gives you Truss + dedicated GPUs; per-minute reflects exclusivity. Modal is true Python Serverless ： per-second billing with sub-second cold starts. Same 输出 (an LLM response), three 不同成本 functions.

This lesson 模型 the six and tells you when each wins.

## 概念

### 三个细分市场

**Custom silicon** ： Groq (LPU), Cerebras (WSE), SambaNova (RDU). Typically 5-10x faster decode than a GPU-based cluster on the same 模型. Higher per-词元 price (Groq was ~$0.99/M on Llama-70B late 2025) but unbeatable for 延迟-sensitive use cases. Groq is 这个生产化 pick for voice agents and real-time translation.

**GPU 平台** ： Baseten, Together, Fireworks, Modal, Anyscale. Run on NVIDIA (H100, H200, B200 in 2026) or sometimes AMD. The economic layer between "raw GPU rental" (RunPod, Lambda) 和 "hyperscaler managed service" (Bedrock).

**API-first marketplaces** ： Replicate, DeepInfra, OpenRouter, Fal. Broad 目录, pay-per-prediction or pay-per-second, emphasize time-to-first-call.

### Fireworks ： 延迟-optimized GPU 平台

- FireAttention engine (custom); marketed as 4x lower 延迟 than vLLM on equivalent configs.
- 批处理 tier at ~50% of Serverless rate for non-interactive 工作负载.
- Fine-tuned 模型 served at the same rate as the base 模型 ： a real 差异点 versus 提供商 that charge a premium for your LoRA.
- Mid-2026: raised 按需 GPU rental $1/小时 effective May 1, 2026. Volume 定价 negotiable at scale.
- Financial signal: $4B valuation, 10T+ 词元/day 处理.

### Together ： breadth-optimized

- 200+ 模型 including open-source releases within days of upstream publication.
- 50-70% 更便宜 than Replicate on equivalent LLM 模型 ： 这个"AI Native Cloud" positioning is volume and 目录.
- Inference + fine-tuning + training in one API.

### Baseten ： 企业级-polish-optimized

- Truss framework: 模型 packaging with dependencies, 密钥, serving config in one manifest.
- GPU range from T4 through B200. Per-minute billing with reasonable cold-start mitigation.
- SOC 2 Type II, HIPAA-ready. Common fintech and healthcare pick.
- $5B valuation, January 2026 Series E ($300M from CapitalG, IVP, NVIDIA).

### Modal ： Python-native-optimized

- 基础设施-as-code in pure Python. Decorate a function with `@modal.function(gpu="A100")` and deploy with one command.
- Per-second billing. Cold starts 2-4s with pre-warming; <1s for small 模型.
- $87M Series B at $1.1B valuation (2025). Strongest developer experience score in independent surveys.

### Replicate ： 多模态 breadth

- Pay-per-prediction. 这个默认 平台 for image, video, and audio 模型.
- Integration ecosystem (Zapier, Vercel, CMS plugins).
- Less competitive on LLM per-词元 rates but wins on 多模态 variety.

### Anyscale ： Ray-native

- Built on Ray; RayTurbo is Anyscale's proprietary inference engine (competes with vLLM).
- Best for distributed Python 工作负载 where the inference 步骤 is one node in a larger graph.
- Managed Ray clusters; tight integration with Ray AIR and Ray Serve.

### 按词元计费与按分钟计费：各自何时胜出

Per-词元 makes sense when 这个工作负载 is 延迟-insensitive and bursty ： you only pay for what you use. Per-minute makes sense when utilization is high and predictable ： you beat per-词元 once you're saturating the GPU.

Rough rule: for 工作负载 above ~30% 持续利用率 of a dedicated GPU, per-minute (Baseten, Modal) starts to beat per-词元 (Fireworks, Together). Below that, per-词元 wins because you avoid paying for 空闲.

### 定制引擎才是真正护城河

Every 平台 above vLLM and SGLang claims a custom engine. FireAttention, RayTurbo, Baseten's inference stack. Custom-engine claims shade marketing ： the honest framing is that vLLM + SGLang represent about 80% of 生产化 open-source inference, and the differentiators at 这个平台 layer are DX, attribution, and SLAs.

### 你应该记住的数字

- Fireworks GPU rental: $1/hr raise effective May 1, 2026.
- Fireworks claim: 4x lower 延迟 than vLLM on equivalent configs.
- Together: 50-70% 更便宜 than Replicate on LLMs.
- Baseten valuation: $5B (Series E, Jan 2026, $300M round).
- Modal valuation: $1.1B (Series B, 2025).
- Per-minute beats per-词元 above ~30% 持续利用率.

```figure
cost-per-token
```

## 使用它

`code/main.py` compares the six vendors on 一个合成 工作负载 跨 定价 模型. Reports $/day and effective $/M 词元. Run it to find 这个盈亏平衡 between per-词元 and per-minute.

## 交付它

This lesson 产出 `outputs/skill-inference-platform-picker.md`. 给定工作负载 画像, SLA, and budget, picks 这个主要推理平台 and names the runner-up.

## 练习

1. Run `code/main.py`. At what 持续利用率 does Baseten (per-minute) beat Fireworks (per-词元) for a 70B 模型 on one H100? Derive the crossover yourself and 比较 to the rule of thumb.
2. Your 产品 serves image generation plus chat plus speech-to-text. Pick 平台 for each modality and name 这个网关 模式 that unifies them.
3. Fireworks raises prices by $1/hr on your 主要模型. 模型 the blended 成本 impact if 40% of your 流量 moves to 批处理 tier (50% off).
4. A regulated 客户 需要 SOC 2 Type II + HIPAA + dedicated GPUs. Which three 平台 are viable and which one wins on FinOps?
5. 比较 成本 per 1,000 predictions for Llama 3.1 70B on Fireworks Serverless, Together 按需, Baseten dedicated, and Replicate API. Which is cheapest at 10 predictions/day? At 10,000?

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| Custom silicon | "non-GPU chips" | Groq LPU, Cerebras WSE, SambaNova RDU ： optimized for decode |
| FireAttention | "Fireworks engine" | Custom attention kernel; marketed at 4x lower 延迟 than vLLM |
| Truss | "Baseten's format" | 模型 packaging manifest; dependencies + 密钥 + serving config |
| Per-词元 | "API 定价" | Charge by 词元 consumed; pay for no 空闲 |
| Per-minute | "dedicated 定价" | Charge by wall-clock GPU time; wins at high utilization |
| Per-prediction | "Replicate 定价" | Charge per 模型 invocation; common for image/video |
| RayTurbo | "Anyscale engine" | Proprietary inference on Ray; competes with vLLM on Ray clusters |
| 批处理 tier | "50% off" | Non-interactive 队列 at reduced rate; common on Fireworks, OpenAI |
| Fine-tuned at base rate | "Fireworks LoRA" | Charge LoRA-served 请求 at base 模型's rate (差异点) |

## 延伸阅读

- [Fireworks Pricing](https://fireworks.ai/pricing) ： per-词元 rates, 批处理 tier, GPU rental.
- [Baseten Pricing](https://www.baseten.co/pricing/) ： per-minute rates, committed 容量, 企业级 tiers.
- [Modal Pricing](https://modal.com/pricing) ： per-second GPU rates and free tier.
- [Together AI Pricing](https://www.together.ai/pricing) ： 模型 目录 and per-词元 rates.
- [Anyscale Pricing](https://www.anyscale.com/pricing) ： RayTurbo and managed Ray 定价.
- [Northflank — Fireworks AI Alternatives](https://northflank.com/blog/7-best-fireworks-ai-alternatives-for-inference) ： comparative assessment.
- [Infrabase — AI Inference API Providers 2026](https://infrabase.ai/blog/ai-inference-api-providers-compared) ： vendor landscape.
