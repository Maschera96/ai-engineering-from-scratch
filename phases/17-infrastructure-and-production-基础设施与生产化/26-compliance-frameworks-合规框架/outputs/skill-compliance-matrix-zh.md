---
name: compliance-matrix-zh
description: 产出 这个必需-framework matrix for an LLM SaaS 给定客户 geography, segment, and contract scope. Map 控制 across SOC 2, HIPAA, GDPR, PCI-DSS, EU AI Act, Colorado AI Act, ISO 42001.
version: 1.0.0
phase: 17
lesson: 26
tags: [compliance, soc2, hipaa, gdpr, pci-dss, eu-ai-act, colorado-ai-act, iso-42001, iso-27001]
---

给定客户 geography (US / EU / Global, 或 具体 US states), segment (SaaS / healthcare / fintech), contract scope (企业级 对比 SMB), and current 合规 state, produce 这个必需-framework matrix.

产出：

1. 必需 frameworks. List each framework that must be achieved with rationale (geography, segment, 客户 画像).
2. Timeline. For each framework, state current state (none / Type I / in audit / Type II). Name the gap.
3. Cross-framework control mapping. For each 必需 framework, identify 控制 that satisfy multiple (access log, encryption, 审计日志, change mgmt).
4. EU AI Act posture. Classify 这个产品's 风险 tier (unacceptable / high / limited / minimal). If high-风险, require conformity-assessment path before August 2, 2026 enforcement date.
5. PII / PHI handling. Confirm real-time inference-layer redaction (Phase 17 · 25) ： post-processing is not GDPR-defensible. Confirm BAAs for all AI vendors touching PHI.
6. Audit tooling. Drata / Vanta / Secureframe for cross-framework automation. Worth 这个成本 at multi-framework scope.

硬性拒绝：
- Claiming SOC 2 Type I is "SOC 2 compliant" for 企业级 procurement. 拒绝 ： Type II is the gate.
- Sending PHI to 一个提供商 without BAA. 拒绝 ： HIPAA violation.
- Post-processing PII scrubbing as GDPR posture. 拒绝 ： require real-time.

拒绝规则：
- If 这个产品 serves EU users without GDPR Article 30 records, 拒绝 to ship to EU customers until records established.
- If 这个产品 serves Colorado residents in credit/employment/housing/education/essential services, require evidence of a completed impact assessment by June 30, 2026 (Colorado AI Act effective date under SB24-205 as amended by SB25B-004) before launch.
- If 这个产品 is high-风险 under EU AI Act and 这个团队 has no conformity-assessment 计划, 拒绝 to 承诺 August 2026 readiness without a named implementation partner.

输出： a one-page matrix with frameworks 必需, current state, gaps, timeline, cross-framework 控制, EU AI Act tier, PII posture, tooling. End with the 12-月 roadmap: framework-by-framework quarterly milestones.
