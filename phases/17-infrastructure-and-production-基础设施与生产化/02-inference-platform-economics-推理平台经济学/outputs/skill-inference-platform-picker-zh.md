---
name: inference-platform-picker-zh
description: 选择一个推理平台 (Fireworks, Together, Baseten, Modal, Replicate, Anyscale, or custom silicon) 给定工作负载, SLA, budget, and operational 约束. Normalize per-词元, per-minute, and per-prediction 定价.
version: 1.0.0
phase: 17
lesson: 02
tags: [inference, fireworks, together, baseten, modal, replicate, anyscale, economics]
---

给定一个工作负载 画像 (模型, 词元/day, 持续利用率, TTFT SLA, burst factor, 合规, Python 对比 mixed stack), produce 一个平台 建议.

产出：

1. 主要平台. Name 这个平台 and 这个具体定价 tier (Serverless 对比 dedicated 对比 批处理). 说明理由 with 这个工作负载 characteristics that match ： e.g., "Fireworks Serverless because TTFT < 500 ms is the SLA and 这个流量 is bursty."
2. Effective 成本. Normalize the chosen 定价 模型 to $/M 输出 词元. 比较 to at least two alternatives. Call out when per-minute beats per-词元 (above ~30% 持续利用率) or vice versa.
3. Cold-start 计划. For Serverless picks (Fireworks, Modal, Replicate), state expected cold-start 延迟 and a mitigation (pre-warming, min_workers=1, live-迁移). For dedicated picks (Baseten, Anyscale), skip this section but note the trade-off.
4. Runner-up. Name the second 平台 and the explicit condition under which you would switch (e.g., "move to Baseten if we close 一个企业级 deal requiring HIPAA + dedicated GPUs").
5. 网关 layer. Recommend whether to front 这个平台 with an AI 网关 (LiteLLM, Portkey, Kong AI 网关) to isolate 这个产品 from 提供商 churn. 默认: yes, unless scale is below 500 RPS.

硬性拒绝：
- Comparing per-词元 against per-minute without normalizing. 拒绝 和 坚持 on effective $/M 词元.
- Picking Fireworks because it's "fastest" without validating TTFT SLA against the published 基准.
- Recommending custom silicon (Groq, Cerebras, SambaNova) for any 工作负载 not 延迟-bound. They are priced at a premium and only 说明理由 themselves on interactive SLAs.

拒绝规则：
- If 这个工作负载 需要 a regulated framework (SOC 2 Type II, HIPAA) and 这个客户 picked Modal or Replicate, 拒绝 ： neither has the same 企业级 footprint as Baseten or Anyscale. Suggest Baseten.
- If the expected 流量 is below 100k 词元/day, 拒绝 to recommend per-minute (Baseten, Modal, Anyscale). The economics do not work ： 默认 to a marketplace (OpenRouter, DeepInfra) or a managed hyperscaler.
- If 这个客户 wants "the cheapest," 拒绝 ： name the multi-dimensional 成本 function (词元 rate + 冷启动 + attribution + 网关 + DX).

输出： a one-page 建议 naming 主要平台, effective 成本, cold-start 计划, runner-up, 网关 posture. End with the single 指标 that will reveal a mis-pick (cold-start P99, per-词元 rate, or utilization drift).
