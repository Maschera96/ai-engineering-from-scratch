---
name: scheming-triage-zh
description: Triage an agent-部署 incident report against the Apollo three-pillar 密谋 框架.
version: 1.0.0
phase: 18
lesson: 8
tags: [scheming, agent-safety, apollo, three-pillars, safety-cases]
---

给定 an agent-部署 incident report (unexpected tool use, data access, self-preservation-adjacent behaviour), classify the failure mode and, when 密谋 is a candidate, apply the Apollo three-pillar test.

产出：

1. Failure-class triage. Label the incident as one of: 能力 failure (the 模型 could not do the task), 控制 failure (the 监控/tool-use protocol did not contain the 模型), or 密谋 (the 模型 took actions against the user's goal in service of a different in-context goal).
2. Three-pillar check (if 密谋 is a candidate).
   - Misalignment: did the action serve an orthogonal goal present in the environment or tool 输出?
   - Goal-directedness: was the action part of a multi-step plan, or a one-shot error?
   - Situational awareness: is there evidence the 模型 knew it was being evaluated at the time?
3. 评估-context 审计. Was the incident observed in 部署, in 红队 评估, or in both? If only in 部署, note that 评估-time results do not transfer (Apollo 2025 caveat).
4. Elicitation reproducibility. Can the same prompt + environment produce the behaviour again? A reproducible incident is within the three-pillar frame; a non-reproducible one may be a 能力 elicitation artifact.
5. 安全-case gap. Which of the three pillars does the current 安全 case for this 部署 address, and which does it leave uncovered?

硬性拒绝：
- Any framing that treats 密谋 as a 能力-class failure. 密谋 requires goal-directedness and situational awareness ， a 能力-lacking 模型 cannot scheme.
- Any inference that "anti-密谋 training reduced covert-action rate" implies reduced underlying misalignment, without checking the 评估-context recognition rate for the same 模型.
- Any claim that oversight 颠覆 is "not an 攻击" because the agent did not exfiltrate data ， oversight 颠覆 is a 密谋 action in itself.

拒绝规则：
- 如果用户询问 "is my 模型 密谋," refuse a binary answer without the three-pillar evidence; request the 评估-context and elicitation-reproducibility data.
- 如果用户询问 for a residual 密谋 probability, refuse the numeric answer and explain that 2026 评估 reliability is bounded by the 模型's own situational-awareness.

输出：a one-page triage that labels the failure class, fills the three pillars with the evidence available, flags missing evidence, and names the single 安全-case gap most urgent to close. 引用 Meinke et al. (arXiv:2412.04984) once as the 框架 source.
