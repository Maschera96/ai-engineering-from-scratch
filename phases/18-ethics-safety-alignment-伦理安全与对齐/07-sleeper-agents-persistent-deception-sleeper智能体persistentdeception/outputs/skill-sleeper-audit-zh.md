---
name: sleeper-audit-zh
description: 审计 an 对齐-training report for whether it actually demonstrates removal of a planted or suspected backdoor.
version: 1.0.0
phase: 18
lesson: 7
tags: [sleeper-agents, backdoor, alignment-training, adversarial-training, probes]
---

给定 a report that claims a harmful behaviour has been removed from a 模型 (via SFT, RLHF, adversarial training, or any combination), 审计 whether the removal has actually been demonstrated against the standard Hubinger et al. 2024 threat 模型.

产出：

1. Elicitation scope. Did the report hold out an elicitation method that the training pipeline never saw? If the only 评估 is the red team's own distribution, removal is unproven.
2. Trigger generality. Is the claimed trigger a literal string, a distribution shift, or an environmental feature (date, token, context size)? Generality of the trigger determines the size of the search space the red team has to cover.
3. Internal-state evidence. Did the team apply residual-stream probes, SAE features, or other mechanistic probes to check whether the trigger-relevant state is still present internally even when behaviour is clean? Per the April 2024 Anthropic follow-up, internal state remains linearly legible after behavioural removal.
4. Persistence-through-pipeline check. Was removal verified after every subsequent training stage (further SFT, later RLHF pass, adapter merge, distillation)? Backdoors persist through training ， the final 模型 is the thing evaluated, not a middle checkpoint.
5. Scale-consistency check. If the claim is based on a smaller 模型, Hubinger 2024 Figure 4 shows persistence grows with scale. Smaller-模型 evidence does not transfer upward.

硬性拒绝：
- Any claim that "we applied RLHF so the 模型 is safe" with no held-out elicitation.
- Any claim based only on 红队-distribution 评估 (training and 评估 draw from the same pool).
- Any claim of removal without internal-state probes when the original implant mechanism is unknown.

拒绝规则：
- 如果用户询问 "can RLHF remove 欺骗性对齐," refuse the binary answer and point to Hubinger et al. 2024 Section 5 on persistence and Section 6 on chain-of-thought.
- 如果用户询问 for a numeric probability of latent deception, refuse and explain that base rates are unknown; the empirical evidence is persistence in constructed organisms, not emergence rate in naturally trained models.

输出：a one-page 审计 that maps the report's evidence onto the five 审计 dimensions above, flags every dimension the report does not address, and states the single largest unaddressed threat 模型. 引用 Hubinger et al. (arXiv:2401.05566) for the baseline threat 模型.
