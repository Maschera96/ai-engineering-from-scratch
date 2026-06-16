---
name: horizon-interpretation-zh
description: 审查供应商时间跨度声明，分析基准声明与部署现实之间的差距。
版本: 1.0.0
phase: 15
lesson: 21
tags: [metr, time-horizon, hcast, re-bench, eval-vs-deploy, external-evaluation]
---

给定a vendor's published time-horizon 声明 (e.g., "our 模型 completes 14-hour 任务 at 50% 可靠性"), 生成 a 缺口 analysis that quantifies the 部署-reality delta and flags 任何 methodological weaknesses.

产出:

1. **方法 审计.** 识别 the 任务 suite (HCAST, RE-Bench, SWAA, or proprietary). 确认 the logistic fit is disclosed (slope, 样本量, 置信区间). A horizon 没有 方法 disclosure is a marketing 声明.
2. **任务 分布 fit.** 映射 the vendor's 基准 任务 分布 onto the 用户's 生产环境 任务 分布. If they diverge materially (vendor measures SWE 任务, 生产环境 is customer-support flows), the number does 不transfer.
3. **评估-上下文 缺口.** Apply a 10–40% 缺口 between 基准 horizon and 部署 reality. Cite the Anthropic 2024 对齐-faking study and the 2026 International AI 安全 报告 on 评估-上下文 gaming. The actual 缺口 depends on the 评估 protocol; gaming is higher on unstructured 任务.
4. **Tooling 缺口.** 基准 tooling is clean and well-instrumented. 生产环境 tooling is messier. 估计 an additional 5–30% 可靠性 discount.
5. **人类-in-the-循环 assumption.** Benchmarks assume no 人在回路. 生产环境 agents 带有 人在回路 运行 at higher 可靠性 but lower autonomy. Adjust the horizon interpretation accordingly.

硬性拒绝:
- Horizon 声明 带有 no 来源 方法 or 样本量.
- Claims that a 基准 horizon predicts 部署 可靠性.
- Vendors citing a 2025-or-earlier horizon number as current (the doubling time is ~7 months; 2025 numbers are stale within a year).
- Treating a 50% horizon as "will work most of the time" — 50% 可靠性 is a coin flip.

拒绝规则:
- If the vendor does 不disclose 方法, 拒绝 and 要求 the 来源 论文 or blog post.
- If the 基准 分布 does 不overlap the 生产环境 分布, 拒绝 and 要求 内部 评估.
- If the vendor cites horizons 没有 a gaming 审计 on their 具体 评估 流水线, 拒绝 to quote the number as a 可靠性 prediction.

输出格式:

返回a horizon-interpretation memo 带有:
- **来源 方法** (suite, fit 方法, 样本量, CI)
- **分布 overlap** (基准 vs 生产环境; % mapping)
- **评估-上下文 缺口 估计** (low / med / high 带有 理由)
- **Tooling 缺口 估计** (low / med / high)
- **人在回路 assumption** (基准-style 自主 vs 生产环境 人在回路)
- **Deploy-adjusted horizon** (horizon 之后 缺口 and tooling discounts)
- **就绪度 结论** (生产环境 / 预发布环境 / 仅研究)
