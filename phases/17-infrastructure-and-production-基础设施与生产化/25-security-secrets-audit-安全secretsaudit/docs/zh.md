# 安全：密钥、API Key 轮换、审计日志、护栏

> Eliminate 密钥 sprawl via centralized vaults (HashiCorp Vault, AWS 密钥 Manager, Azure Key Vault). Never store credentials in config files, env files in VCS, spreadsheets. Use IAM roles over static keys; OIDC for CI/CD. The AI-网关 模式 is the 2026 solution: apps → 网关 → 模型 提供商, 搭配 网关 pulling credentials from vault at runtime. Rotate in vault and all apps pick up in minutes ： no redeploys, no Slack "who has the new key" messages. Rotation 策略 ≤90 days; scan with TruffleHog / GitGuardian / Gitleaks on every commit. Zero-trust: MFA, SSO, RBAC/ABAC, short-lived 词元, device posture. PII scrubbing uses entity recognition to mask PHI/PII before forwarding; consistent tokenization (Mesh approach) maps sensitive values to stable placeholders so the LLM preserves code/关系 semantics. Network egress: LLM services in dedicated VPC/VNet subnet whitelisting only `api.openai.com`, `api.anthropic.com` etc; block all other outbound. The 2026 事故 driver: Vercel supply-chain attack via compromised CI/CD credentials exfiltrated env vars across thousands of 客户 deployments.

**Type:** Learn
**Languages:** Python（标准库， 玩具 PII-scrubber + audit-log writer)
**Prerequisites:** Phase 17 · 19 (AI Gateways), Phase 17 · 13 (Observability)
**Time:** ~60 分钟

## 学习目标

- Enumerate the four 密钥-management anti-patterns (config files in VCS, hardcoded env, spreadsheets, static keys) and name their replacements.
- 解释 the AI-网关-pulls-from-vault 模式 as 2026 生产化 standard.
- Implement a PII scrubber with consistent tokenization (same value → same placeholder) so semantics survive.
- Name the 2026 Vercel supply-chain 事故 and what it taught about CI/CD credential hygiene.

## 问题

An intern commits `.env` with API keys. They delete it quickly. The keys are already in git history ： GitGuardian scan catches it, your rotation process is "Slack 这个团队, update 40 config files, redeploy all services." 8 hours later, half your services are live and half are waiting for deploy windows.

Separately, user 提示词 include "My SSN is 123-45-6789." 提示词 goes to OpenAI. You have a BAA but your internal 策略 is to mask PII before forwarding. You didn't.

Separately, your EKS cluster's LLM pod can reach any internet host. Someone exfils data via DNS lookup to an attacker-controlled domain. Nothing blocked it.

安全 for LLM services has to address all three vectors. Vault-backed credentials. PII scrubbing. Network egress filtering. 审计日志.

## 概念

### Centralized vault + IAM-role pull

**Vault**: HashiCorp Vault, AWS 密钥 Manager, Azure Key Vault, GCP 密钥 Manager. One source of truth.

**IAM role**: app/网关 authenticates via its IAM identity, not a static key. Vault returns 这个密钥 for the lifetime of 这个词元.

**The AI-网关 模式**: 网关 pulls `OPENAI_API_KEY` from vault at 请求 time. Rotate in vault; next 请求 gets the new key. No redeploys.

### Rotation 策略 ≤ 90 days

All API keys, vault root 词元, CI/CD credentials. Automated rotation where possible. Manual rotation logged and tracked.

### 密钥扫描

- **TruffleHog** ： regex + entropy on commits.
- **GitGuardian** ： commercial, high accuracy.
- **Gitleaks** ： OSS, runs in CI.

运行on every commit. Block PR if new 密钥 detected.

### 零信任姿态

- MFA 必需 on all accounts.
- SSO via SAML/OIDC.
- RBAC (role-based) or ABAC (attribute-based) for fine grained access.
- Short-lived 词元 (hours, not days).
- Device posture ： only corp devices with disk encryption.

### PII / PHI scrubbing

Before 这个提示词 leaves your infra:

1. Entity recognition (spaCy NER, Presidio, commercial).
2. Mask matched entities: `"My SSN is 123-45-6789"` → `"My SSN is [SSN_TOKEN_A3F]"`.
3. Consistent tokenization (Mesh approach): same value maps to the same placeholder so the LLM preserves relationships.
4. Optional reverse mapping for LLM response.

Static regex filters catch basic patterns; NER catches more. Use both.

### 输入 + 输出 护栏

输入: block known jailbreaks, forbidden topics; 限流 per-user.

输出： regex scrub for leaked 密钥 (API key patterns, email patterns in refusal contexts), classifier for 策略 violations.

### Network egress whitelist

LLM services in a dedicated subnet:
- Whitelist: `api.openai.com`, `api.anthropic.com`, vector DB endpoints, vault endpoints.
- Everything else: drop.
- DNS via allowlist-only resolver (avoid DNS-tunneling exfil).

### 审计日志

Immutable log of every LLM call with:
- Timestamp.
- User / tenant.
- 提示词 hash (not raw 提示词 for 隐私).
- 模型 + version.
- 词元 counts.
- 成本.
- Response hash.
- Any guardrail trips.

Retain per regulatory requirement (SOC 2 1 year, HIPAA 6 years).

### The 2026 Vercel 事故

Supply-chain attack: compromised CI/CD credentials exfiltrated env vars across thousands of 客户 deployments. Lesson: CI/CD credentials are prod-equivalent. Store in vault. Scope narrowly. Rotate aggressively.

### 你应该记住的数字

- Rotation 策略: ≤ 90 days.
- Scan on every commit: TruffleHog / GitGuardian / Gitleaks.
- Vercel 2026: CI/CD creds compromised → thousands of 客户 env vars leaked.
- 审计日志 retention: SOC 2 = 1 year, HIPAA = 6 years.

## 使用它

`code/main.py` implements a toy PII scrubber with consistent tokenization and an append-only 审计日志.

## 交付它

This lesson 产出 `outputs/skill-llm-security-plan.md`. Given regulatory scope and current state, plans the vault 迁移, scrubber, egress, 审计日志.

## 练习

1. Run `code/main.py`. Send two 提示词 referencing the same SSN. Confirm both get the same placeholder.
2. Design the network egress 策略 for a vLLM-on-EKS 部署 calling OpenAI + Anthropic + Weaviate.
3. You 发现 a key in git history (2 years old). What's the correct response ： rotate the key, scrub history, or both? 说明理由.
4. Your 审计日志 grows 10 GB/day. Design retention tiers (hot 30d, warm 12mo, cold 6yr).
5. Argue whether reverse-tokenization (substituting real values back into LLM response) is worth the complexity versus keeping placeholders visible.

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| Vault | "密钥 store" | Centralized credential management service |
| IAM role | "identity-based auth" | Role assumed by app; returns short-lived creds |
| OIDC for CI/CD | "cloud-issued 词元" | No static keys in CI ： identity via OIDC |
| TruffleHog / GitGuardian / Gitleaks | "密钥 scanners" | Commit-time 密钥 detection |
| RBAC / ABAC | "access control" | Role-based 对比 attribute-based |
| PII scrubbing | "data masking" | Remove or tokenize sensitive entities |
| Consistent tokenization | "stable placeholders" | Same value → same 词元 each time |
| Mesh approach | "Mesh tokenization" | Semantic-preserving tokenization 模式 |
| Egress whitelist | "outbound allowlist" | Only permitted domains reachable |
| 审计日志 | "immutable history" | Append-only record for 合规 |

## 延伸阅读

- [Doppler — Advanced LLM Security](https://www.doppler.com/blog/advanced-llm-security)
- [Portkey — Manage LLM API keys with secret references](https://portkey.ai/blog/secret-references-ai-api-key-management/)
- [Datadog — LLM Guardrails Best Practices](https://www.datadoghq.com/blog/llm-guardrails-best-practices/)
- [JumpServer — Secrets Management Best Practices 2026](https://www.jumpserver.com/blog/secret-management-best-practices-2026)
- [Microsoft Presidio](https://github.com/microsoft/presidio) ： PII detection and anonymization.
- [HashiCorp Vault docs](https://developer.hashicorp.com/vault/docs)
