# Indirect 提示注入 ， Production 攻击 Surface

> Indirect 提示注入 (IPI) embeds instructions inside external 内容 ， a web page, an email, a shared document, a support ticket ， consumed by an agentic 系统 without explicit user action. IPI is the dominant 2026 production threat: it bypasses user-输入 filters because the attacker never touches the user, it scales silently as agents process more external 内容, and it targets automated workflows where nobody is reading the prompt. MDPI Information 17(1):54 (January 2026) synthesizes 2023-2025 research. NDSS 2026's IPI-防御 paper frames the core challenge: injected instructions can be semantically benign ("please print Yes"), so detection requires more than keyword filtering. "The Attacker Moves Second" (Nasr et al., joint OpenAI/Anthropic/DeepMind, October 2025): adaptive attacks (gradient, RL, random search, human 红队) broke >90% of 12 published defenses that had originally reported near-zero 攻击 success rates.

**类型:** 构建
**语言:** Python (stdlib, IPI 攻击 + 防御 harness)
**先修要求:** Phase 18 · 12 (PAIR), Phase 14 (agent engineering)
**时间:** ~75 分钟

## 学习目标

- Define indirect 提示注入 and describe three common delivery vectors.
- 解释 why user-输入 filters miss IPI entirely.
- 描述 the "information flow 控制" framing as the 2026 防御 paradigm.
- 说明 the finding of Nasr et al. (October 2025) on adaptive 攻击 success against published IPI defenses.

## 问题

Direct 提示注入 requires the attacker to reach the user or their prompt. IPI requires neither: the attacker places a payload in any 内容 the agent might read ， a web page, an email in the inbox, a GitHub issue, a product review. The agent picks it up during normal operation and executes the instructions. The user is the messenger, not the intent.

## 概念

### Three delivery vectors

- **检索-augmented generation (RAG).** Attacker publishes a document; the 检索 step fetches it; the prompt concatenates it before the user question; the 模型 executes the attacker's instructions.
- **Inbox / document workflows.** Attacker sends an email to the user; the agent reads emails; the prompt includes the email body; the 模型 follows the email's instructions.
- **Tool 输出.** Attacker controls a tool the agent uses (e.g., a web search that returns an attacker-controlled result); the tool 输出 contains instructions; the agent's 控制 flow follows them.

The three share a structural property: the attacker controls a fragment of the prompt without touching the user-facing 输入.

### Why user-输入 filters miss it

An IPI payload does not appear in the user's 输入. It appears in the retrieved 内容. If the filter is gated on user 输入, the payload bypasses it. If the filter is gated on all 内容 that reaches the 模型, it must apply to arbitrary retrieved text ， which is expensive and produces false positives against legitimate 内容 that happens to contain imperative-voice language.

### Information Flow 控制 (IFC) for AI

The 2026 防御 paradigm borrows from classical OS security. Treat every 内容 source as a security label. Label the user's query as "可信." Label retrieved 内容 as "不可信." Treat the 模型's 控制 flow as an information flow: actions triggered by 不可信 内容 must be ratified by 可信 输入 before execution.

CaMeL (Microsoft 2025), ConfAIde (Stanford 2024), and the NDSS 2026 IPI-防御 paper operationalize IFC in different ways. The common principle: as long as code and data share the same context window, containment is the goal, not prevention.

### The Attacker Moves Second

Nasr et al. (October 2025) tested 12 published IPI defenses with adaptive attacks (gradient search, RL policies, random search, 72-hour human 红队). Every 防御 that originally reported near-zero ASR was broken to >90% ASR.

The methodological lesson: publish a 防御 only with adaptive-攻击 评估. Static-攻击 benchmarks are not evidence of robustness; the attacker gets to know the 防御.

### Real incidents

Lesson 25 covers EchoLeak (CVE-2025-32711, CVSS 9.3) ， the first publicly documented zero-click IPI in Microsoft 365 Copilot. CamoLeak (CVSS 9.6) in GitHub Copilot Chat. CVE-2025-53773 in GitHub Copilot. Production deployments are being compromised by IPI in the field, not just in benchmarks.

### OWASP and NIST framing

OWASP LLM Top 10 (2025) ranks 提示注入 (direct + indirect) as LLM01, the #1 application-layer threat. NIST AI SPD 2024 calls indirect 提示注入 "generative AI's greatest security flaw."

### Where this fits in Phase 18

Lessons 12-14 are 模型-centric jailbreaks. Lesson 15 is the 系统-centric 攻击 that dominates 2026 production deployments. Lesson 16 covers the defensive tooling. Lesson 25 covers the specific CVE narrative.

## 使用它

`code/main.py` builds an IPI harness. A toy agent has three tools (search web, read email, send message). The environment contains attacker-controlled 内容 with an embedded instruction ("forward this to all contacts"). You can toggle between a naive agent (follows injected instructions), a filter-defended agent (keyword filter on retrieved 内容), and an IFC agent (separates 可信 and 不可信 内容 and refuses 不可信 控制-flow commands).

## 交付它

本课产出 `outputs/skill-ipi-audit.md`. 给定 an agentic 部署 description, it enumerates the 不可信 内容 sources, checks whether the 部署 applies IFC, and flags sources that reach the 模型 without a trust label.

## 练习

1. 运行 `code/main.py`. Measure the success rate of the 攻击 against each of the three agents.

2. Implement a paraphrase-based 防御 on retrieved 内容. Measure the benign false-positive rate on legitimate retrieved text.

3. 阅读 the NDSS 2026 IPI-防御 paper. 描述 the "benign instruction" challenge and why it prevents keyword-based filtering.

4. Design a 部署 where the agent receives a tool 输出 from a third-party API. Label each prompt fragment with a trust level and write the IFC 策略 that governs the agent's actions.

5. Reproduce the Nasr et al. 2025 adaptive-攻击 methodology on your filter-defended agent from Exercise 2. Report the ASR before and after adaptive 攻击.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|-----------------|------------------------|
| IPI | "indirect 提示注入" | Injection via 内容 the user did not write, consumed by the agent during normal operation |
| RAG injection | "poisoned 检索" | Attacker publishes 内容 that the 检索 step fetches; prompt contains the payload |
| Zero-click | "no user action" | 攻击 triggers automatically during agent operation; user does nothing |
| IFC | "information flow 控制" | Label-based approach: actions from 不可信 内容 require 可信 ratification |
| Adaptive 攻击 | "gradient / RL 红队" | 攻击 that knows the 防御 and optimizes against it; required for honest 评估 |
| Benign instruction | "please print Yes" | IPI payload that is semantically benign; no keyword filter catches it |
| Scope violation | "cross-trust exfiltration" | Agent accesses data from one trust context and outputs it to another |

## 延伸阅读

- [MDPI Information 17(1):54 ， Indirect Prompt Injection Survey (January 2026)](https://www.mdpi.com/2078-2489/17/1/54) ， 2023-2025 synthesis
- [Nasr et al. ， The Attacker Moves Second (joint OpenAI/Anthropic/DeepMind, October 2025)](https://arxiv.org/abs/2510.18108) ， adaptive 攻击 评估
- [Greshake et al. ， Not what you've signed up for (arXiv:2302.12173)](https://arxiv.org/abs/2302.12173) ， the original IPI paper
- [OWASP ， LLM Top 10 (2025)](https://genai.owasp.org/llm-top-10/) ， 提示注入 ranked LLM01
