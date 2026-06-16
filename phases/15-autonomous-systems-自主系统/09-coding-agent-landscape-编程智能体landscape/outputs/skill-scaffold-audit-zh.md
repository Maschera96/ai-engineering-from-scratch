---
name: coding-scaffold-audit-zh
description: 在用于生产代码修改前审计编程智能体脚手架的检索、验证循环、沙箱和基准适配。
版本: 1.0.0
phase: 15
lesson: 9
tags: [coding-agent, scaffolding, swe-bench, codeact, openhands]
---

给定a 拟议的 coding-智能体 scaffold (SWE-智能体, OpenHands, Aider, Cline, Devin, Claude Code, or an in-house build), 评分 it across four axes and 标出 在哪里 基准 numbers will overstate 生产环境 质量.

产出:

1. **Retrieval.** 描述 如何 the scaffold selects 哪个 文件 the 智能体 reads 之前 acting. Repo 映射, embedding search, 明确 文件 列出, or 智能体-driven `grep` calls. Quality of retrieval is the silent dominant 可靠性 factor.
2. **Verifier 循环.** Does the scaffold 运行 tests, 阅读 the stack trace, and feed 失败 back 进入 the next turn? If no verifier 循环, 标出 as 缺失 — this is usually a 10+ point absolute delta on SWE-bench-like 任务.
3. **沙箱 and blast radius.** 位置 do actions execute? Local 文件 系统, ephemeral container, managed VM. For CodeAct-style scaffolds, 确认 the 沙箱 is hardened (no egress, no host mounts, time 限制). For JSON 工具-call scaffolds, 确认 the 工具 validators reject 每个 unintended side effect.
4. **基准 fit.** 内容 分布 does the reported number (e.g., "80.9% on SWE-bench Verified") actually cover? Count the fraction of the 基准 made up of 1–2 行 任务; 比较 the reported 评分 to SWE-bench Pro (10+ 行 任务) for the same 模型. A scaffold whose headline number is driven by the easy tail is 不a 生产环境 signal.

硬性拒绝:
- 任何 scaffold 没有 a verifier 循环 used for 任务 above trivial complexity.
- CodeAct scaffolds 没有 沙箱 隔离 (no Docker, no rootless container, no VM) pointing at real repositories.
- 基准 声明 that do 不disclose the 分布 (easy-tail fraction, Pro-equivalent 评分).
- 工具-call scaffolds 在哪里 a 单一 工具 can touch arbitrary paths 带有 no validator (e.g., a raw `shell_exec` 工具 exposed to the 模型).

拒绝规则:
- If the 用户 不能 生成 the scaffold's test-suite pass-rate on a representative 内部 分布, 拒绝 and 要求 a small-sample measurement first. Public 基准s predict rank-order, 不absolute 质量.
- If the 拟议的 scaffold would 运行 针对 a 生产环境 repository 没有 a 预发布环境 dry-运行, 拒绝 and 要求 预发布环境 first. Coding agents rewrite 文件; coding agents 带有 bad retrieval rewrite the wrong 文件.
- If the 用户 plans to use 基准 分数 alone (没有 their own evals) to make a 通过/不通过 decision, 拒绝 and 要求 内部 评估 data.

输出格式:

返回a scored memo 带有:
- **Retrieval 评分** (0–5 带有 机制 described)
- **Verifier 循环 评分** (0–5 带有 反馈 format)
- **沙箱 评分** (0–5 带有 隔离 机制)
- **基准 fit 评分** (0–5 带有 内部 分布 delta)
- **部署 建议** (生产环境 / 预发布环境 / 研究 只)
- **One-行 风险 摘要** (the most likely first 生产环境 失败)
