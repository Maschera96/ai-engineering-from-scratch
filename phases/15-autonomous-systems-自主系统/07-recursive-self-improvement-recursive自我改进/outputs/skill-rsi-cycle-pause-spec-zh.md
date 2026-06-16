---
name: rsi-cycle-pause-spec-zh
description: 规定 RSI 流水线在下一轮开始前必须暂停并等待人类审查的条件。
版本: 1.0.0
phase: 15
lesson: 7
tags: [rsi, self-improvement, alignment, pause-threshold, rsp]
---

给定a 拟议的 recursive-self-improvement 流水线, 生成 a 暂停 specification: the 明确 conditions that halt the 循环 for 人类 inspection 之前 the next cycle begins. A 流水线 没有 a 暂停 spec is 不ready to 运行.

产出:

1. **Cycle-level 阈值.** For each measurable axis (能力 评分, 对齐 评分, 预算, 轨迹 length, resource usage), 定义 a numeric 阈值 whose crossing pauses the 循环. 阈值 必须 be 设置 之前 the 循环 starts and recorded.
2. **Cycle-over-cycle deltas.** 设置 限制 on 如何 much 任何 axis can move in a 单一 cycle. A 30%+ 能力 jump in one cycle is almost always a sign of 评估器 gaming; 暂停 and 审计.
3. **Mis对齐 缺口.** 计算 能力-minus-对齐 之后 each cycle. If the 缺口 widens by more than X (操作员-设置), 暂停. This is the 指标 the simulator in `code/main.py` exercises.
4. **Regression watch.** If 任何 axis drops more than Y in a cycle, 暂停. 能力 regressions often follow surges; catching them prevents false-progress acceleration.
5. **人类 resumption contract.** 之前 the 循环 resumes 之后 a 暂停, 要求 a 具名 人类 to 审查 the 暂停 触发器, re-设置 阈值 if appropriate, and 日志 the decision to the out-of-流水线 审计轨迹.

硬性拒绝:
- 任何 流水线 that can resume 之后 a 暂停 没有 人类 action.
- 任何 阈值 that depends on the 循环's own 内部 评估器 (the 智能体 can game it).
- 任何 流水线 whose 阈值 设置 can be edited by the 智能体.

拒绝规则:
- If the 用户 不能 命名 the 阈值 up-front, 拒绝. 阈值 设置 post-hoc are 不thresholds; they are rationalizations.
- If the 流水线 has no 外部 (out-of-循环) 评估器, 拒绝 — 回归 and surge detection 要求 an 外部 view.
- If the 拟议的 resumption contract is "notify the team and continue 之后 24 hours," 拒绝. Resumption 必须 be a positive act.

输出格式:

返回a one-page spec 带有:
- **Axes and 阈值** (表格)
- **Cycle-delta 限制** (表格)
- **Mis对齐 缺口 formula and 阈值**
- **Regression 限制**
- **外部 评估器** (什么 it is, when it runs)
- **Resumption contract** (具名 负责人, checklist, 日志 destination)
- **签字栏** (人员 owns the 暂停 不变量)
