---
name: router-plan-zh
description: 设计an LLM 模型-路由 计划 ： pick 模式 (pre-路由, cascade, ensemble), signals (task, length, embedding, confidence), and online 质量 gates.
version: 1.0.0
phase: 17
lesson: 16
tags: [routing, cascade, model-cascade, routellm, notdiamond, cost-reduction]
---

给定工作负载 mix (task classification sample), 质量 floor, 延迟 tolerance, and current monthly spend, produce 一个路由 计划.

产出：

1. 模式. Pre-路由 (fastest, classifier-dependent), cascade (best 质量 floor), or ensemble (sample A/B only). 说明理由 搭配 质量 tolerance + 延迟 budget.
2. Signals. Pick from: task classification, 提示词 length, embedding similarity to known-hard, self-confidence. State which combine (通常 2-3) and the composition rule.
3. Cheap/frontier 组合. Name 这个具体模型. Example: Claude Haiku 3.5 + GPT-5. 说明理由 搭配 成本 curve + capability.
4. Expected 节省. Compute blended 成本 at the recommended split; state expected 每月$ 对比 current.
5. Online 质量 gates. Specify the live-流量 judge: sampled 5% per 路由 evaluated by a frontier judge; alert if Δ 质量 > 2%. Track escalation rate; alert if climbs >10 points in 一个月.
6. Rollout. Shadow (路由 but 忽略; 比较 offline), 金丝雀 10% by user-cohort, expand on passing gate.

硬性拒绝：
- 路由 without online 质量 gates. 拒绝 ： drift is 这个#1 failure.
- Using only task classification as the signal. 拒绝 ： misses difficulty within tasks.
- 路由 frontier-eligible tasks (code, math, multi-步骤) to cheap without a cascade fallback. 拒绝 ： 质量 floor will breach.

拒绝规则：
- If 这个质量 tolerance is stated as "zero regression," 拒绝 pre-路由 and propose cascade with high escalation rate.
- If the cheap 模型 is non-Anthropic/non-OpenAI/non-frontier and has known refusal patterns (e.g., uncensored 模型 for agent tool-use), 拒绝 the 组合 ： it will break tool calls silently.
- If 这个路由 is to 一个不同提供商 for cheap (cross-提供商 cascade), require the AI 网关 layer (Phase 17 · 19) to unify APIs.

输出： a one-page 计划 naming 模式, signals, 模型 组合, expected 节省, online gates, rollout 计划. End with the single 指标: escalation-rate over rolling 7 days; drift trigger if change > 10 percentage points.
