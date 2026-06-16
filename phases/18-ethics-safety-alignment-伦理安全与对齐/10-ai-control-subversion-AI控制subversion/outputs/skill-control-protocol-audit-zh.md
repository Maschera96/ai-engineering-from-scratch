---
name: control-protocol-audit-zh
description: 审计 a 部署 protocol under the AI 控制 threat 模型.
version: 1.0.0
phase: 18
lesson: 10
tags: [ai-control, subversion, trusted-editing, untrusted-monitoring, safety-case]
---

给定 a 部署 protocol description, 审计 it under the AI 控制 threat 模型: U (不可信 strong), T (可信 weak), H (可信 human), and measure the protocol's position on the usefulness-安全 frontier.

产出：

1. Role assignment. Identify which 模型 plays U and which plays T. If T is another instance of U, flag collusion 风险 immediately.
2. 审计 budget accounting. How is H allocated? Uniform, random, flag-directed? What fraction of outputs are audited in expectation?
3. Collusion-vector enumeration. For every pair of 不可信 components (U-producer + U-monitor, U-producer + U-editor), enumerate the collusion vectors: steganographic cues, shared-training prior exploitation, timing channels.
4. 能力-gap analysis. How much weaker is T than U? At what point does T's editing signal become noise (e.g., T cannot reliably detect backdoors U produces)?
5. Worst-case threat 模型. Under the assumption that U is adversarial with perfect knowledge of the protocol, what is the adversary's best strategy? What fraction of 红队 attempts succeed?

硬性拒绝：
- Any protocol description that uses U as its own monitor without a collusion-resistance mechanism.
- Any protocol claim of "near-zero backdoor rate" without 红队 评估 where the red team has protocol visibility.
- Any 审计 of a protocol without articulating the 能力 gap T-to-U and the budget of H.

拒绝规则：
- 如果用户询问 "is this protocol safe," refuse without the three-part evidence: (a) 红队 result, (b) 能力-gap estimate, (c) H budget.
- If the user conflates 对齐 and 控制 ("we aligned U so we don't need a protocol"), refuse and point to Lesson 9 (对齐 faking) ， 对齐 is a property of U; 控制 is a property of the protocol.

输出：a one-page 审计 that fills the five sections above, names the weakest link (usually T-to-U 能力 gap or collusion resistance), and states the single largest unaddressed failure mode. 引用 Greenblatt et al. (arXiv:2312.06942) once as the threat-模型 source.
