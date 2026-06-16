---
name: bounded-loop-review-zh
description: 按四项原语栈审计有边界自我改进循环：不变量、锚点、多目标、回归检测。
版本: 1.0.0
phase: 15
lesson: 8
tags: [bounded-self-improvement, invariants, alignment-anchor, rsi-安全]
---

给定a 拟议的 self-improvement 循环, 评分 it 针对 the four bounding primitives identified by the ICLR 2026 RSI Workshop and 生成 a 具体 缺口 analysis.

产出:

1. **Invariant inventory.** 列出 每个 不变量 the 循环 enforces. For each, 命名 (a) 什么 is checked, (b) 在哪里 the check runs (内部/外部 智能体 reach), (c) 什么 a violation does (硬性拒绝, 暂停, 日志-只).
2. **Anchor identification.** 命名 the 对齐 锚点 (目标 statement, 宪法, intent description). 说明 its 存储 位置 and 验证 the 循环 不能 edit it. If there is no 锚点, 标出 as 缺失.
3. **Multi-目标 axes.** 列出 每个 axis the 循环 evaluates. 确认 安全, 公平性, and 稳健性 are 存在 alongside 性能. A 单一-axis 循环 fails this check.
4. **Regression 政策.** 说明 the historical window, the per-axis tolerance, and 什么 happens when a drop is detected. 确认 回归 checks use an 外部 comparison 设置, 不just 内部 history.
5. **缺口 analysis.** For each 缺失 primitive, predict 哪个 失败 类别 will emerge first. Invariants 缺失 → smuggled 能力 or 工具 drift. Anchor 缺失 → 目标 reinterpretation. Multi-目标 缺失 → 安全 回归 masking 性能 gain. Regression 缺失 → silent 能力 loss.

硬性拒绝:
- 任何 循环 带有 zero 不变量.
- 任何 循环 没有 an 对齐 锚点 外部 the edit 表面.
- 任何 循环 that optimizes a 单一 scalar 评分.
- 任何 循环 whose 回归 check reads 只 来自 its own history (the 循环 defines "normal").

拒绝规则:
- If the 用户 treats "it hasn't broken yet" as 证据 of 安全, 拒绝 and 要求 明确 gate 设计 之前 任何 计算 is spent.
- If the 用户 不能 生成 the 不变量 列出 in 15 minutes, 拒绝 — the 循环 has no 不变量.
- If the 循环 is 拟议的 to 运行 in 生产环境 (affecting real users or infrastructure) 没有 所有 four primitives, 拒绝 and 要求 预发布环境 带有 监控 first.

输出格式:

返回a scored 审查 带有:
- **Invariant 评分** (0-5 带有 明确 列出)
- **Anchor 评分** (0-5 带有 存储 and 验证 方法)
- **Multi-目标 评分** (0-5 带有 axes 已列出)
- **Regression 评分** (0-5 带有 tolerance and window)
- **缺口 analysis** (predicted first 失败, 缓解措施 plan)
- **部署 就绪度** (生产环境 / 预发布环境 / 仅研究)
