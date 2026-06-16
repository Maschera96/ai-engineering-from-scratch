---
name: llm-security-plan-zh
description: 产出 an LLM 安全 计划 covering 密钥 vault, PII scrubbing with consistent tokenization, network egress allowlist, 审计日志 retention, and zero-trust posture.
version: 1.0.0
phase: 17
lesson: 25
tags: [security, vault, hashicorp, aws-secrets-manager, pii, presidio, egress, audit-log, zero-trust, ci-cd-supply-chain]
---

给定regulatory scope (SOC 2, HIPAA, GDPR), current credential state, and network/egress posture, produce 一个安全 计划.

产出：

1. Vault 迁移. Pick vault (HashiCorp, AWS 密钥 Manager, Azure Key Vault, GCP 密钥 Manager). 网关 模式: apps → 网关 → vault at runtime. Deprecate hardcoded env and config-file credentials.
2. 密钥 scanning. Enable TruffleHog / GitGuardian / Gitleaks on every commit. Block PR on detection.
3. Rotation 策略. ≤ 90 days. Automated where possible. Dedicated rotation for CI/CD credentials (shorter ： 30d recommended).
4. PII scrubbing. Entity recognition (Presidio + regex). Consistent tokenization (same value → same placeholder) to preserve semantics.
5. Egress allowlist. Whitelist LLM 提供商 domains, vector DB, vault endpoints. DNS allowlist resolver.
6. 审计日志. Append-only, immutable. 必需 fields: user, tenant, 提示词/response hash, 词元, 成本, guardrail trips. Retention per framework (SOC 2 1y / HIPAA 6y).
7. CI/CD hygiene. OIDC identity federation (no static cloud keys). Scope CI/CD credentials narrowly. Cite the 2026 Vercel supply-chain 事故 as motivation.

硬性拒绝：
- Static keys in config files. 拒绝.
- Storing raw 提示词 in 审计日志. 拒绝 ： hash only unless the regulatory framework explicitly 需要 otherwise.
- Allowing egress to `*` 或 "the internet." 拒绝 ： whitelist.

拒绝规则：
- If no vault is acceptable to 这个客户 (air-gapped requirement), 拒绝 normal 计划 and design a file-based-with-rotation fallback. Explicitly note it is less secure.
- If PII scrubbing is declined for "延迟" reasons, 拒绝 ： 这个延迟 is typically <20 ms and the regulatory 风险 dwarfs it.
- If rotation >90 days is requested for a vault root 词元, 拒绝 ： it becomes a breach vector.

输出： a one-page 计划 with vault, scanning, rotation, scrubbing, egress, 审计日志, CI/CD posture. End with the single 指标: 密钥-scan hit count per 月; target zero.
