# 作为降本基础能力的模型路由

> A dynamic broker evaluates every 请求 (task type, 词元 length, embedding similarity, confidence) and sends 简单 queries to a cheap 模型, escalating complex ones to a frontier 模型. Also called 模型 cascading. 生产化 case studies show 20-60% 成本 reduction at iso-质量 across US/UK/EU deployments; a 30% 路由 efficiency improvement on high-volume SaaS turns into six-figure annual 节省. The 2026 context is that LLM inference prices dropped ~10x per year ： a GPT-4-class 词元 went from $20/M to ~$0.40/M from late 2022 to 2026. Most of the drop is better serving stacks (Phase 17 · 04-09), not hardware. 路由 is how you convert that price drop into margin without 产品 regression. The failure mode is cheap-模型 drift: 这个路由 pushes 40% to a weaker 模型, 质量 drops 3-5% on 推理 tasks, no one notices for a quarter. Gate routes by online 质量 指标, not just offline eval sets.

**Type:** Learn
**Languages:** Python（标准库， 玩具 cascading router模拟器)
**Prerequisites:** Phase 17 · 01 (Managed LLM Platforms), Phase 17 · 19 (AI Gateways)
**Time:** ~60 分钟

## 学习目标

- 解释 模型 cascading: cheap-first with confidence check, escalate on low confidence.
- Enumerate the four 路由 signals (task classification, 提示词 length, embedding similarity to known-hard set, self-confidence from first-pass).
- Compute expected blended 成本 at target 路由 split and 质量 loss tolerance.
- Name the drift-监控 指标 (online 质量 gate) that catches cheap-模型 creep.

## 问题

Your service costs $80k/月 on GPT-5. Your analytics show 70% of queries are 简单: "what time is it in Paris?" "rephrase this sentence." A Haiku-class 模型 handles those perfectly at 3% of 这个成本. 30% need GPT-5's 推理 ： coding, math, multi-步骤 planning.

如果you 路由 the 70% to cheap and 30% to expensive, your bill drops ~65% at the same 产品 质量. This is 路由. The trick is building the broker without regressing 质量.

## 概念

### 四种路由信号

1. **Task classification**: 简单/complex/codegen/math/chat. Can be a rules-based classifier, a small LLM (Haiku-class at $0.25/M), or embedding similarity to labeled buckets. 输出: 路由 = cheap / balanced / frontier.

2. **提示词 length**: 提示词 >4K 词元 often need frontier for coherence. 提示词 <500 词元 通常 don't.

3. **Embedding similarity to known-hard set**: if the query is close (cosine > 0.88) to a known-hard bucket, escalate to frontier 直接.

4. **Self-confidence from first-pass**: send to cheap; if 模型's log-probs show low confidence OR it refuses OR 输出 hedging language, retry on frontier. Adds P95 延迟 on ~10% of 流量 but saves 50%+ on the other 90%.

### 三种模式

**Pre-路由** (classifier up front): ~5-10ms 延迟 新增; fastest overall.

**Cascade** (cheap-first, escalate on low confidence): ~1.2x 中位数 延迟 (cheap run plus verify), ~2x on escalated. Best 质量 floor.

**Ensemble 路由** (run cheap and frontier in parallel for a sample, reward-模型 pick): highest 质量, highest 成本; use only for 关键 A/B.

### 实现

AI gateways (Phase 17 · 19) expose 路由. LiteLLM has `router` config with fallback and 成本-路由. Portkey has guards + 路由. Kong AI 网关 has plugin-based 路由. OpenRouter's 模型市场 exposes 一个建议 API.

Open-source: RouteLLM (LMSYS), Not Diamond (commercial), 提示词 Mule.

### 2026 年价格曲线

| 模型 class | Late 2022 | 2026 | Change |
|-------------|-----------|------|--------|
| GPT-4-level 质量 | ~$20/M | ~$0.40/M | 50x 更便宜 |
| Frontier (GPT-5, Claude 4) | ： | ~$3-10/M | new tier |

Most of the improvement is serving efficiency ： the core lessons in Phase 17 · 04-09 turned into 提供商-side 成本 drops. 路由 lets you capture those gains at the app layer instead of waiting for all your users to migrate to the cheap tier.

### 漂移才是真正风险

Your 路由 sends 40% to the cheap 模型. Over six months, the task distribution shifts (users get more sophisticated, ask longer questions). 这个路由器 doesn't notice because its classifier was trained on Q1 data. 质量 drops silently. Nobody complains loud enough. You find out in a competitor 基准 you lost.

Gate routes by online 质量 指标:

- User thumbs-up / thumbs-down per 路由.
- Automated LLM-judge on a held-out sample (5%) per 路由.
- Escalation rate: if cascade is kicking up-路由 >30%, the cheap 模型 is being over-routed.
- Refusal rate per 路由.

### 你应该记住的数字

- 2026 路由 节省 at iso-质量: 20-60% case studies.
- LLM price drop 2022-2026: ~10x per year aggregate.
- GPT-4-level 2022 对比 2026: ~$20/M → ~$0.40/M.
- Cascade 延迟 impact: ~1.2x 中位数, ~2x escalated (~10% of 流量).

## 使用它

`code/main.py` simulates pre-路由, cascade, and ensemble on a mixed 工作负载. Reports blended 成本, 质量 loss, and escalation rate.

## 交付它

This lesson 产出 `outputs/skill-router-plan.md`. 给定工作负载 和 质量 budget, picks 一个路由 模式 and signals.

## 练习

1. Run `code/main.py`. At what accuracy floor does cascade beat pre-路由?
2. Your user base is 30% 企业级 (complex queries), 70% free tier (简单). Design 这个路由 split. What online 指标 gates it?
3. 一个路由 drops 质量 by 2% but saves 40%. Is that a ship? Depends on 产品 ： argue both.
4. Implement a confidence check using logprobs from OpenAI / Anthropic APIs. What's the threshold you start with?
5. Over six months, escalation rate climbs from 8% to 22%. Diagnose three causes and the fix for each.

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| 模型 路由 | "成本 broker" | Dynamic choice of 模型 per 请求 |
| 模型 cascade | "cheap-first escalate" | Run cheap, fall through to frontier on low confidence |
| Pre-路由 | "classify first" | Classifier up front; no re-run |
| Ensemble 路由 | "parallel pick" | Run multiple, reward-模型 picks best |
| Escalation rate | "uprouted %" | Fraction of cascade 请求 that escalated |
| RouteLLM | "LMSYS 路由器" | OSS 路由器 library |
| Not Diamond | "commercial 路由器" | SaaS 模型-路由 产品 |
| Drift | "cheap creep" | Distribution shift without 路由器 noticing |
| Online 质量 gate | "live check" | Automated LLM-judge sampling live 流量 |

## 延伸阅读

- [AbhyashSuchi — Model Routing LLM 2026 Best Practices](https://abhyashsuchi.in/model-routing-llm-2026-best-practices/)
- [Lukas Brunner — Rise of Inference Optimization 2026](https://dev.to/lukas_brunner/the-rise-of-inference-optimization-the-real-llm-infra-trend-shaping-2026-4e4o)
- [RouteLLM paper / code](https://github.com/lm-sys/RouteLLM)
- [Not Diamond — model routing](https://www.notdiamond.ai/)
- [OpenRouter](https://openrouter.ai/) ： multi-模型 网关 搭配 路由 primitives.
