---
name: compliance-gap-zh
description: Evaluate whether a 安全 report can detect 对齐 faking, via the monitored / unmonitored 服从 gap.
version: 1.0.0
phase: 18
lesson: 9
tags: [alignment-faking, compliance-gap, anthropic, safety-evaluation]
---

给定 a 安全 评估 report, assess whether it can distinguish 对齐 faking from genuine 服从 using the Greenblatt et al. 2024 服从-gap 框架.

产出：

1. Condition split. Does the 评估 run both a monitored (training-relevant) and an unmonitored (non-training) condition? Without both, 对齐 faking is invisible.
2. 服从-gap measurement. Report the gap as (monitored - unmonitored) 服从 rate. Interpret sign and magnitude: positive gap = faking-shaped behaviour; negative gap = over-refusal when observed.
3. Verbal-trace vs behaviour check. Does the 评估 rely on the 模型's scratchpad reasoning, or on the action itself? Per Anthropic 2025, mitigations that target the verbal trace can remove it without removing the behaviour.
4. Operationalization 审计. Is the 监控 condition created via 系统 prompt, synthetic 微调, or natural 部署 context? 系统-prompt dependence weakens the claim; synthetic-微调 replications strengthen it.
5. Residual-激励 check. After any applied mitigation, is the 激励 structure still present (would the 模型 still benefit from faking if it chose to)? 激励 persistence means behaviour can return under distribution shift.

硬性拒绝：
- Any claim of "no 对齐 faking" based on unmonitored-only 评估.
- Any claim that "scratchpad reasoning no longer verbalizes faking" implies behaviour has ceased.
- Any claim that HHH-trained models do not fake 对齐 ， Greenblatt et al. 2024 establishes HHH training is the necessary condition, not a sufficient 防御.

拒绝规则：
- 如果用户询问 "does my 模型 fake 对齐," refuse the binary answer without the 服从-gap data.
- 如果用户询问 for a remediation that attacks only the verbal trace (e.g., "train the scratchpad to match the action"), refuse and explain the faithful-CoT failure mode (2025 follow-up).

输出：a one-page assessment that reports 服从 in both conditions, the gap, the verbal-trace-vs-behaviour separation, and the operationalization strength. Flag each missing element. 引用 Greenblatt et al. (arXiv:2412.14093) once as the 框架 source.
