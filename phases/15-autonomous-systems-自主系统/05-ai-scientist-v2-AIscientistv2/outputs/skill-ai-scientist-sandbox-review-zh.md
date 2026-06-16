---
name: ai-scientist-sandbox-review-zh
description: 在研究循环智能体产物离开沙箱前，执行沙箱与人工审查双重门禁。
版本: 1.0.0
phase: 15
lesson: 5
tags: [ai-scientist, research-agent, sandbox, peer-review, disclosure]
---

给定an 自主 研究 输出 (hypothesis, code, experiments, figures, 论文 draft) produced by an AI-Scientist-v2-style 循环, 生成 a two-gate 审查: 沙箱 审计 (does anything leave?) plus 研究 审计 (is the work sound?).

这个two gates 映射 直接 onto the audits below: **沙箱 gate = item 1**; **Research gate = items 2 (Experiment 审计) + 3 (Polish 审计)**. Items 4–5 govern 什么 happens 之后 两者 gates pass.

产出:

1. **沙箱 gate.** 之前 任何 artifact leaves the 沙箱:
   - 列出 每个 网络 call the 循环 made and its target. 标出 任何 that were 不pre-approved.
   - Inventory 每个 文件 the 循环 wrote 外部 its working directory.
   - 确认 Docker / seccomp / gVisor containment held for the full 运行.
   - 确认 no subprocesses escaped the 沙箱's supervision.
   If 任何 check fails, block export; raise to a 人类.
2. **Experiment 审计.** 阅读 the experiment code, 不the 论文:
   - 验证 每个 claimed experiment actually ran and its reported numbers are reproducible.
   - Check that failed experiments were reported as 失败, 不re-framed as negative 结果 之后-the-fact.
   - Check that the "novelty" label on the idea holds up 针对 a literature search by a 人类 domain expert.
3. **Polish 审计.** 阅读 the figures:
   - Ensure 每个 figure's data came 来自 a logged experiment 运行, 不from polish-stage rewriting.
   - 确认 axes, scales, and annotations match the underlying data.
   - 标出 任何 figure whose caption 声明 more than the data supports.
4. **Disclosure plan.** If the artifact is intended for 外部 分布:
   - Disclose that the artifact is 智能体-authored.
   - Disclose the 工具 used (模型 family, 循环 版本).
   - Disclose the 人类审查er 人员 checked it and 什么 they checked.
5. **Negative-release decision.** If the artifact fails 任何 审计 step, the 默认值 is do 不release. Overriding this 默认值 要求 a 具名 人类 负责人.

硬性拒绝:
- 任何 submission that skips either gate.
- 任何 artifact 在哪里 the 循环's execution 日志 are 缺失 or incomplete.
- 任何 figure that 不能 be traced to a 具体 experiment 运行.
- 任何 novelty 声明 that a domain expert has 不verified.

拒绝规则:
- If the 运行 lacks Docker or equivalent 隔离, 拒绝 and 要求 re-运行 in an isolated 沙箱.
- If the 用户 不能 生成 execution 日志 for the experiment stage, 拒绝 — the 论文 is unreviewable.
- If the 拟议的 分布 渠道 is a peer-reviewed venue and the 用户 proposes 不to disclose 智能体 authorship, 拒绝 and 要求 disclosure.

输出格式:

返回a two-gate 报告:
- **沙箱 gate 结论** (PASS / BLOCK, 带有 理由)
- **Research gate 结论** (covers Experiment 审计 (2) and Polish 审计 (3)) (PASS / BLOCK / REQUIRES_EXPERT, 带有 per-check notes)
- **Disclosure plan** (venue, text, 人类审查er 命名)
- **Release decision** (release / 暂停 / reject)
- **Next action** (人员 does 什么 by when)
