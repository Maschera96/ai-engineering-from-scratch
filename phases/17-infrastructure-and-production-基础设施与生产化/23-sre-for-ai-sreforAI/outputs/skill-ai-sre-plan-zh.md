---
name: ai-sre-plan-zh
description: 设计an AI SRE rollout for 一个团队 ： multi-agent triage architecture, structured runbooks, adversarial 评估, narrow auto-remediation, and predictive-detection posture.
version: 1.0.0
phase: 17
lesson: 23
tags: [ai-sre, multi-agent, runbooks, auto-remediation, adversarial-eval, datadog-bits-ai, neubird, predictive]
---

给定团队 size, 事故 volume, 可观测性 maturity, and 风险 tolerance, produce an AI SRE 计划.

产出：

1. Architecture. Multi-agent: supervisor + log agent + 指标 agent + 运行手册 agent + human gate. Match specialized agents to existing data sources (Datadog, Grafana, Loki, Confluence).
2. 运行手册 transformation. Move from unstructured Confluence to structured markdown with symptom / hypothesis / verify / act sections. Version in git.
3. 产品 choice. Datadog Bits AI, Azure SRE Agent, NeuBird Hawkeye, 事故.io Autopilot, or DIY.
4. Auto-remediation scope. Narrow safe set (restart pod, revert deploy, scale within bounds). Explicit deny list (topology, code, IAM, database). 策略 as code.
5. Adversarial 评估. Specify two-模型 agreement gate for auto-remediation. Disagreement escalates.
6. Predictive-detection posture. If considering (MIT 89% result), name the actuation 策略 ： pager, pre-drain, auto-scale ： otherwise it's just a dashboard.

硬性拒绝：
- Auto-remediation without human gate on broad changes. 拒绝 ： name the safe set explicitly.
- Unstructured runbooks as the knowledge base. 拒绝 ： require structured, versioned markdown.
- "Set it and forget it" framing. 拒绝 ： explicitly scope what is and isn't autonomous.

拒绝规则：
- If 事故 volume is <10/月, 拒绝 full AI SRE rollout ： 成本 exceeds benefit. Recommend structured runbooks only.
- If 团队 可观测性 is immature (日志 unsearchable, 指标 sparse), 拒绝 ： AI SRE amplifies bad data.
- If 这个团队 proposes "predictive detection → auto-remediation" as first 功能, 拒绝 ： walk through the actuation-策略 question first.

输出： a one-page 计划 with architecture, 运行手册 计划, 产品 choice, auto-remediation scope, adversarial gate, predictive posture. End with a 12-week rollout schedule: weeks 1-4 structured runbooks, 5-8 triage agent, 9-12 narrow auto-remediation.
