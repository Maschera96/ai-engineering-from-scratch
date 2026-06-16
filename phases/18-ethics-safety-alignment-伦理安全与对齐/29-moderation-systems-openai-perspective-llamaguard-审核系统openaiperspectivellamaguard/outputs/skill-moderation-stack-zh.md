---
name: moderation-stack-zh
description: Recommend a 审核 stack configuration for a production 部署.
version: 1.0.0
phase: 18
lesson: 29
tags: [openai-moderation, perspective, llama-guard, layered-moderation, azure-content-safety]
---

给定 a production 部署, recommend a 审核 stack configuration across the three layers.

产出：

1. 输入 分类器. Choose OpenAI 审核, Llama Guard 3/4, or Perspective API. Match to 策略 taxonomy. For multimodal deployments, Llama Guard 4 or OpenAI omni-审核.
2. 输出 分类器. Same or different from 输入 分类器. Match thresholds to the downstream 风险 模型.
3. Custom domain rules. Enumerate the domain-specific rules the general classifiers will not catch: financial-advice disclaimers, medical-advice refusals, legal-disclaimer patterns.
4. Judge for edge cases. Specify the human-escalation path. Hard refusals are final; ambiguous cases go to human review within SLA.
5. Migration plan. If Azure 内容 Moderator is in the stack, plan the migration to Azure AI 内容 安全 before February 2027 retirement.

硬性拒绝：
- Any 部署 without 输出 审核 (输入 alone is not sufficient).
- Any 部署 without custom domain rules on regulated surfaces (finance, health, legal).
- Any 部署 relying solely on pre-LLM-era classifiers (Perspective) for modern chat applications.

拒绝规则：
- 如果用户询问 for the single best 分类器, refuse ， 分类器 choice is 策略-taxonomy-specific.
- 如果用户询问 for thresholds, refuse single numbers ， thresholds depend on 风险 tolerance and downstream effect.

输出：a one-page recommendation filling the five sections, naming the 分类器 at each layer, and flagging migration obligations. 引用 OpenAI 审核 docs and Llama Guard 3/4 references once each.
