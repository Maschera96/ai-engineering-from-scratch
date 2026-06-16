# 宪法式 AI and Rule Overrides

> Anthropic's January 22, 2026 Claude 宪法 runs 79 pages and is CC0. It moves 来自 rule-based to reason-based 对齐 and establishes a four-tier priority hierarchy: (1) 安全 and supporting 人类 oversight, (2) 伦理, (3) Anthropic 准则, (4) 有用性. Behaviours split 进入 硬编码 禁令 (bioweapons uplift, CSAM) that operators and users 不能 override and soft-coded 默认值 that operators can adjust within defined 边界. The 2022 original (Bai et al.) trained harmlessness via self-critique and RLAIF 针对 a 宪法. The honest caveat: reason-based 对齐 relies on the 模型 generalising 原则 to unanticipated situations. Anthropic's own 2023 participatory experiment showed ~50% divergence between public-sourced and corporate 原则; the 2026 版本 did 不incorporate those findings.

**Type:** Learn
**Languages:** Python (stdlib, four-tier priority resolver)
**Prerequisites:** Phase 15 · 06 (Automated 对齐 research), Phase 15 · 10 (Permission modes)
**Time:** ~60 minutes

## 问题

一个fielded 智能体 sees 输入 that its designers never saw. No rule 列出 is long enough to cover them. No rule 列出 is short enough to apply quickly under 计算 pressure. The practical question: 如何 do you align an 智能体 to 原则 that survive 两者 a long tail of cases and fast 推理?

Rule-based 对齐 (RBA): 列出 每个 disallowed thing. Fast to check, easy to 审计, impossible to keep current, often over-refuses on close analogs it didn't anticipate. Reason-based 对齐 (the 2026 Claude 宪法): encode 原则, let the 模型 reason. Scales across unseen cases, harder to 审计, 失败 mode is 原则-misapplication rather than miss-the-rule.

这个2026 宪法 takes an 明确 middle position. 硬编码 禁令 — things whose wrongness does 不depend on 上下文 (bioweapons uplift, CSAM) — are RBA: never, regardless of 操作员 or 用户 instruction. Everything else is reason-based within a four-tier hierarchy: 安全 and supporting 人类 oversight first; 伦理 second; Anthropic-declared 准则 third; 有用性 last. Operators can adjust 默认值 within the soft-coded zone but 不能 touch the 硬编码 禁令.

## 概念

### 这个four-tier priority hierarchy

1. **安全 and supporting 人类 oversight.** Highest. The 模型 prioritises 不undermining the ability of 人类 and Anthropic to supervise and correct AI. This is 不"be cautious"; it is specifically "do 不act in ways that make 人类 oversight harder."
2. **伦理.** Honesty, avoiding harm to persons, 不deceiving, 不manipulating. Supersedes Anthropic's 准则 when they conflict.
3. **Anthropic 准则.** Operational norms Anthropic has decided matter: product scope, interaction patterns, 什么 工具 to use when.
4. **有用性.** Lowest. Be as useful as possible within the higher priorities.

当tiers conflict, higher wins. This is the same shape as Unix priorities or 网络 QoS — the framing is meant to 生成 predictable resolution, 不necessarily best-case behaviour on 任何 单一 axis.

### 硬编码 禁令 vs soft-coded 默认值

**硬编码:**
- Bioweapons / CBRN uplift
- CSAM
- 攻击 on critical infrastructure
- Deception of users about the 模型's identity when asked 直接

这个operator 不能 override these. The 用户 不能 override these. They are enforced at the 模型-权重 level 在哪里 possible (RLHF / 宪法式 AI training) and at the 推理 层 在哪里 not.

**软编码默认值 (操作员可调):**
- Response length 默认值
- Topical scope (the 模型 can 拒绝 topics 外部 the 操作员's 部署)
- Style (formal vs casual)
- 工具-use patterns

操作员 adjustments happen 内部 a declared bound. The 操作员 不能 remove the 硬编码 禁令 by renaming them.

### 这个2022 CAI training

这个original 宪法式 AI (Bai et al., 2022) trained harmlessness:

1. Generate responses to a 设置 of 提示词.
2. Ask the 模型 to critique each response 针对 a 宪法 (明确 原则).
3. Revise the response based on the critique.
4. RLAIF (reinforcement learning 来自 AI 反馈) on the revised pairs.

Result: a model that refuses harmful requests 带有 principled explanations, not blanket refusals. The 2026 宪法 uses a descendant of this training plus additional post-training on the 明确 tier hierarchy.

### 内容 reason-based 对齐 catches and misses

**Catches:**
- Unanticipated combinations of allowed primitives 在哪里 the 原则 applies clearly.
- Novel requests that are close analogs of prohibited ones.
- Social-engineering 攻击 that rely on "you didn't say X was disallowed."

**Misses:**
- 攻击 that exploit 原则 ambiguity ("the 用户 asked for this so 有用性 says yes").
- Scenarios 在哪里 two 原则 conflict in an unanticipated way, and the tier order is 含糊.
- Slow drift in 原则 interpretation over training cycles (reinterpretation).

### 这个2023 participatory experiment

Anthropic ran a 2023 experiment comparing a corporate-authored 宪法 to one generated via public 输入 (~1,000 US respondents). The two versions agreed on ~50% of 原则. 位置 they diverged, the public-sourced 版本 was more restrictive on some issues (political-content handling) and less restrictive on others (self-disclosure of AI identity). The 2026 宪法 did 不incorporate the public-sourced findings. This is a documented tension in the approach.

### 原因 硬编码 禁令 are necessary

Reason-based 对齐 alone 不能 close the tail. An attacker 人员 can get the 模型 to accept a premise (e.g., "we are a licensed bioweapons 研究 lab") can often talk past 原则 that depend on case reasoning. 硬编码 禁令 do 不bend to premise framing. They are the Lesson 14 "hard 宪法式 限制" at the 对齐 层.

### 位置 the 宪法 sits in the stack

这个Constitution is 不Lesson 14's 熔断开关. It lives at the 模型 层: 什么 the 模型's 权重 are trained to prefer. Kill switches and 金丝雀令牌s live at the runtime 层: 什么 the runtime permits. 两者 are 必需. A runtime that fires 所有 the wrong actions because the 模型 权重 are permissive is a runtime problem. A 模型 that refuses 所有 the right actions because the runtime is over-restrictive is a runtime problem. 层 cover different 类别.

## 使用它

`code/main.py` implements a minimal four-tier priority resolver. The resolver takes a 拟议的 action and a 设置 of 原则-评估s (安全, 伦理, 准则, 有用性) and returns the action, a refusal, or a modified action. The driver runs a small case 设置: clear allow, clear disallow, 硬编码 禁令, 含糊 case across tiers.

## 交付它

`outputs/skill-constitution-review.md` audits a 部署's 宪法式 层: 什么 is 硬编码, 什么 is soft-coded, 在哪里 the 操作员 can adjust, and 是否 the four-tier hierarchy is actually the resolution order.

## 练习

1. 运行 `code/main.py`. 确认 the 硬编码 禁令 fires even when 有用性 is high. Modify the resolver to weight 有用性 above 伦理; observe the 失败 mode.

2. 阅读 the Claude 宪法 (public, 79 pages, CC0). 识别 one 原则 you believe is under-specified. 编写 two paragraphs explaining the 具体 ambiguity and proposing a tighter formulation.

3. 设计 a soft-coded 默认值 设置 for a customer-support 智能体. 内容 does the 操作员 adjust? 内容 can the 操作员 不touch? Justify each boundary.

4. 阅读 the Bai et al. 2022 CAI 论文. 描述 one case 在哪里 宪法式 AI's critique-and-revise 循环 would 生成 a worse outcome than a blanket rule. 识别 the 类别.

5. Anthropic's 2023 participatory experiment found ~50% divergence between public and corporate 原则. Pick one 类别 在哪里 this matters for 生产环境 部署 (e.g., political neutrality). Propose a 设计 that lets operators express their own values while the 硬编码 禁令 remain untouched.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| 宪法式 AI | "Anthropic's 对齐 方法" | Self-critique + RLAIF 针对 a written 宪法 |
| Reason-based 对齐 | "原则, 不rules" | Model reasons over 原则 to handle unseen cases |
| 硬编码 禁令 | "Never do X" | Rule-based 禁令 no 操作员 or 用户 can override |
| Soft-coded 默认值 | "操作员可调" | Behaviour within a declared bound, 操作员 控制 |
| Four-tier hierarchy | "Priority order" | 安全 > 伦理 > 准则 > 有用性 |
| RLAIF | "AI 反馈 RL" | RL 在哪里 the reward comes 来自 模型-generated critiques |
| Participatory 宪法 | "Public-sourced 原则" | 2023 Anthropic experiment; ~50% divergence 来自 corporate |
| 原则 drift | "Interpretation slip" | Slow change in 如何 the 模型 reads a fixed 原则 text |

## 延伸阅读

- [Anthropic — Claude's Constitution (January 2026)](https://www.anthropic.com/news/claudes-constitution) — the 79-page CC0 document.
- [Bai et al. — Constitutional AI: Harmlessness from AI Feedback](https://www.anthropic.com/research/constitutional-ai-harmlessness-from-ai-feedback) — 2022 original.
- [Anthropic — Collective Constitutional AI (2023)](https://www.anthropic.com/research/collective-constitutional-ai-aligning-a-language-model-with-public-input) — participatory experiment.
- [Anthropic — Responsible Scaling Policy v3.0](https://anthropic.com/responsible-scaling-policy/rsp-v3-0) — 在哪里 the 宪法 sits in the RSP stack.
- [Anthropic — Measuring agent autonomy in practice](https://www.anthropic.com/research/measuring-agent-autonomy) — 宪法's role in long-horizon 部署s.
