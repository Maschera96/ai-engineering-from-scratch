---
name: horizon-reality-check-zh
description: 检查一个任务是否适合交给长周期智能体，给出时间跨度、可靠性和控制措施的现实评估。
版本: 1.0.0
phase: 15
lesson: 1
tags: [自主-agents, metr, time-horizon, reliability, deployment]
---

给定a 拟议的 自主 任务 (什么 the 智能体 应该 do, 如何 long a 人类 expert would take, 什么 the 失败 成本 is), 生成 a reality check on 是否 the current 前沿模型's horizon actually covers it.

产出:

1. **Expert-time 估计.** Ask the 用户 for the median expert completion time in minutes or hours. If they 不能 估计 it, 拒绝 and redirect them to measure a small sample first.
2. **Headroom 比例.** Divide the chosen 模型's 50% METR horizon by the expert-time 估计. 标出 任何 比例 under 4x — at 50% success probability, you want a generous margin. At 比例 2x or below, 拒绝 the 部署 unless 人在回路 is in the 循环 on 每个 significant action.
3. **Reliability 预算.** 估计 轨迹 length in 工具 calls, then 计算 end-to-end success at per-step 可靠性 0.95, 0.99, 0.995. If the 任务 length exceeds the 50%-success 阈值 at your assumed per-step 可靠性, 要求 检查点s or split the 任务.
4. **评估-vs-deploy adjustment.** Apply a 20-40% 缺口 between 基准 horizon and deploy-上下文 horizon. Cite the Anthropic 2024 对齐-faking study or the 2026 International AI 安全 报告 when justifying to stakeholders.
5. **必需 控制.** Based on headroom, 列出 the minimum 设置 of 控制: 预算 上限, iteration 上限, 熔断开关, 人在回路 检查点 points, 金丝雀令牌s, and 轨迹 审计 schedule.

硬性拒绝:
- 任何 部署 at horizon 比例 below 2x 没有 人在回路 on 每个 consequential action.
- 任何 声明 that a 模型 "can do" a 任务 based on the METR horizon alone. The horizon is the 50% mark on a logistic curve; tail 失败 are guaranteed.
- Treating METR horizons as a floor rather than a ceiling.

拒绝规则:
- If the 用户 不能 估计 expert-time for the 任务, 拒绝 and ask them to measure a small sample first. Anything else is guesswork.
- If the 拟议的 任务 would 成本 more than the 用户's worst-case 预算 at full 模型 pricing, 拒绝 and recommend 预算 控制 来自 Lesson 13 之前 proceeding.
- If the 用户 describes a 任务 that touches irreversible actions (financial transactions, 生产环境 database writes, emails to customers) 没有 任何 人在回路 层, 拒绝. The horizon argument does 不clear irreversible 部署.

输出格式:

返回a short memo 带有:
- **任务 摘要** (一句话)
- **Expert-time 估计** (带有 units)
- **Headroom 比例** (带有 明确 number)
- **End-to-end 可靠性 估计** (表格 at three per-step rates)
- **Minimum 控制** (项目符号)
- **通过 / 暂停 / 不通过** (明确 结论 plus one-sentence justification)
