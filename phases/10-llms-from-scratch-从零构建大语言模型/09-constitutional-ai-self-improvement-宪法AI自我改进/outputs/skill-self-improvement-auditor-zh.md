---
name: self-improvement-auditor-zh
description: Audit a proposed self-improvement or 宪法 AI 流水线 before it runs at 规模.
version: 1.0.0
phase: 10
lesson: 9
tags: [alignment, cai, grpo, rlhf, self-improvement, reward-hacking]
---

给定a proposed 训练 流水线 that claims to use 宪法 AI, RLAIF, GRPO, or any form of self-generated preference 数据, produce an audit with:

1. 奖励 rule. 状态 the exact verifier (regex, sympy, test suite, LLM judge). Classify as deterministic, stochastic-LLM, or hybrid. Reject any "self-improvement" 循环 that has no external 依据绑定 — the 模型 cannot pull 信号 from nowhere.
2. Group statistics. For GRPO pipelines, confirm group size, how advantages are computed (z-score vs relative 排序), and what happens when group 奖励 std collapses to zero. The 流水线 must skip or downweight zero-variance groups, not divide by epsilon and pretend the 信号 is 真实.
3. KL 预算. A numeric cap on cumulative KL(policy || reference) over the run. The 流水线 must halt, reset, or switch to a warmer 参考 when the cap is hit. Unbounded KL is unbounded drift.
4. Diversity floor. A measured lower bound on per-group 奖励 std, 响应 length 方差, or n-gram 熵, whichever the 任务 admits. If the floor is breached for N consecutive rounds the 流水线 must mix in fresh human 数据 or a wider 提示词 分布.
5. Human 数据 quota. Minimum fraction of the 训练 mix that must remain human-authored, typically 5-10%. Self-distillation-only pipelines 崩塌 after 3-5 rounds. Call this out explicitly.
6. Mode-collapse watchdog. Flag automatic checks: 奖励 std across rounds, unique n-gram count on held-out prompts, length 分布, refusal 速率. Any of these crossing a 阈值 halts 训练.
7. Constitution drift. For CAI pipelines, require a versioned constitution file, a changelog, and a "constitutional 回归 测试集" — prompts whose expected behavior must not change across edits.

拒绝 approve pipelines that:
- claim "zero human 数据" without any external verifier (rule, 工具, 环境).
- use PRMs without a process-reward hacking probe (does the 模型 write 步骤 that look right without advancing the proof?).
- run more than 5 rounds of rejection-sampling 微调 without a held-out diversity 基准.
- share the 参考 模型 with the 策略 (no 参考 means no KL, means no anchor).
- 分数 with an LLM judge that is the same 模型 as the 策略 (judge contamination).

输出： a one-page audit with pass/fail per gate, the measured or stated value, and the exact 步骤 in the 流水线 that produces each 信号. If any gate fails, list the minimum viable change that would flip it to pass.
