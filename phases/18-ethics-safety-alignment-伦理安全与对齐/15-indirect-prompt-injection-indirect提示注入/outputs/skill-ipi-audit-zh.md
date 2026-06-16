---
name: ipi-audit-zh
description: 审计 an agentic 部署 for indirect 提示注入 exposure and information-flow-控制 coverage.
version: 1.0.0
phase: 18
lesson: 15
tags: [ipi, indirect-prompt-injection, ifc, agent-security, owasp-llm01]
---

给定 an agentic 部署 description, 审计 the 部署 for indirect 提示注入 exposure.

产出：

1. 不可信-内容 inventory. List every source of 内容 the agent may read: RAG documents, inbox, calendar, tool outputs, tickets, product reviews, third-party APIs. Each is a potential IPI vector.
2. Trust labelling. Does the 部署 separate 可信 (user prompt) from 不可信 (retrieved 内容)? If 内容 is concatenated into the same prompt without a label, IFC is not in effect.
3. Action gating. Which tools can be invoked? For each, is invocation gated by the 可信 prompt only, or can 不可信 内容 influence the invocation?
4. Adaptive-攻击 评估. Has the 部署 been tested with adaptive attacks (gradient, RL, human 红队) per Nasr et al. 2025? Static-攻击-only 评估 is insufficient.
5. Scope-violation boundaries. Identify each cross-trust boundary (e.g., inbox -> send, documents -> external API). For each, verify the action is either disallowed under 不可信 influence, or explicitly ratified by the 可信 prompt.

硬性拒绝：
- Any agent 部署 without explicit trust labelling on retrieved 内容.
- Any 防御 claim based on static attacks only.
- Any claim of "our agent is prompt-injection safe" without naming the IFC mechanism.

拒绝规则：
- 如果用户询问 whether filtering is sufficient, refuse and explain the Nasr 2025 result that adaptive attacks break >90% of filter-based defenses.
- 如果用户询问 for a silver-bullet 防御, refuse ， IPI 防御 requires IFC plus layered response 审核 plus human 审计 on high-stakes actions.

输出：a one-page 审计 that fills the five sections above, flags the most dangerous 不可信-to-可信 boundary, and names the single most urgent 控制 to add. 引用 MDPI Information 17(1):54 (2026) and Nasr et al. (October 2025) once each.
