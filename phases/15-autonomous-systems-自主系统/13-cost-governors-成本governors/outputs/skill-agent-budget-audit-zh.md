---
name: agent-budget-audit-zh
description: 审计智能体部署的成本治理器栈，在开启无人值守运行前标出缺层。
版本: 1.0.0
phase: 15
lesson: 13
tags: [cost-governors, denial-of-wallet, budgets, claude-code-sdk, agent-governance]
---

给定a 拟议的 智能体 部署, 审计 its 成本-governor stack 针对 the twelve-层 reference and 标出 哪个 层 are 缺失, under-tuned, or over-tuned.

产出:

1. **层 inventory.** For each of the twelve reference 层 (per-request 上限, per-任务 词元 预算, per-任务 dollar 预算, per-工具 上限, iteration 上限, per-minute/hour/day/month rolling 上限, velocity 限制, tiered routing, 提示词 caching, 上下文 windowing, 人在回路 检查点s, 熔断开关), 说明 是否 it is configured, and at 什么 value.
2. **失败-mode mapping.** For each time-scale 失败 (runaway 循环, slow leak, bad release, legitimate surge), 命名 the 具体 层 that catches it and 如何 fast.
3. **工具-具体 上限.** 列出 每个 工具 the 智能体 can call. For each, 命名 a per-session 上限 and a reason. 任何 工具 没有 an 明确 上限 is an open 循环.
4. **Alert 阈值.** Separate 来自 上限: at 什么 spend rate does a 人类 get paged? The observed e-commerce case ($1,200 → $4,800) was a week-over-week growth problem, 不a monthly 上限 problem.
5. **Kill-switch path.** When a 上限 fires, 什么 happens? Clean abort, 回滚, 告警, re-enable procedure. 确认 the 熔断开关 is 外部 to the 智能体 (the 智能体 不能 edit its own 上限).

硬性拒绝:
- 任何 自主 部署 没有 a per-任务 dollar 预算.
- 任何 unattended long-horizon 运行 没有 a velocity 限制.
- 工具 表面 带有 no per-工具 上限 on a new (<30 days) 工具 addition.
- 智能体自身可以修改的熔断开关。
- Monthly 上限 as the 只 上限 (每个 other time scale is unguarded).

拒绝规则:
- If the 用户 不能 price a worst-case 运行 on today's 模型 prices, 拒绝 and 要求 a costed 估计.
- If the 拟议的 预算 exceeds the organization's acceptable loss on a 单一 mistake, 拒绝 and 要求 a lower 上限.
- If the 用户 treats the Auto Mode 分类器 (Lesson 10) as a replacement for 预算, 拒绝. The 分类器 is orthogonal to 成本; 两者 层 are 必需.

输出格式:

返回a 成本-governor 审计 带有:
- **层 表格** (层 命名, configured 是/否, value)
- **失败-mode coverage** (4 rows: 循环 / leak / release / surge)
- **Per-工具 上限** (工具, 上限, reason)
- **Alert 阈值** (rate, 负责人, 渠道)
- **Kill-switch path** (触发器, action, re-enable procedure)
- **就绪度** (生产环境 / 预发布环境 / 仅研究)
