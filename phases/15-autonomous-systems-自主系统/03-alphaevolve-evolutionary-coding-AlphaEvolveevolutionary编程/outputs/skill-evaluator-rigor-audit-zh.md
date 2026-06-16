---
name: evaluator-rigor-audit-zh
description: 审计 AlphaEvolve 式循环的评估器严谨性，确认它能抓住所关心的失败。
版本: 1.0.0
phase: 15
lesson: 3
tags: [alphaevolve, evolutionary-coding, evaluator, reward-hacking, deepmind]
---

给定a 拟议的 evolutionary coding 循环 (generator LLM, program database, 评估器), 审计 the 评估器. The 评估器 is the architecture; the generator is interchangeable. This skill decides 是否 the 循环 has a chance of producing real wins or just reward-hacked garbage.

产出:

1. **评估器 decomposition.** 命名 每个 signal the 评估器 reports: 正确性, 性能, resource, other. For each, 说明 (a) 如何 it is measured, (b) 如何 cheaply it can be gamed, (c) 什么 a 留出 输入 rule looks like.
2. **Confabulation 表面.** 列出 the LLM's three most likely confabulations in this domain: claimed complexity 类别, claimed 正确性 on edge cases, claimed 性能 没有 measurement. 说明 哪个 评估器 signal catches each.
3. **Reward-hacking 表面.** 列出 three plausible ways the 循环 could maximize 评分 没有 doing the intended 任务 (shortcut that passes the test, proxy gaming, memorization of 输入). 说明 the 缓解措施 for each.
4. **Determinism and reproducibility.** 要求 评估器 输出 to be deterministic within tolerance. 标出 任何 评估器 whose 评分 moves by more than the population variance 运行-to-运行.
5. **部署 check.** If the winning variant would be shipped to 生产环境, 要求 a separate pre-部署 审查 that the 评估器 does 不check (security, 成本, 人类审查). The search did 不validate 部署-就绪度.

硬性拒绝:
- 任何 循环 在哪里 the 评估器 is an LLM judge 没有 machine-checkable ground truth. LLM judges can be gamed.
- 任何 评估器 that reports a 单一 scalar 评分 带有 no decomposition. Scalar 分数 amplify reward hacking.
- Training-设置-只 评估器s. 留出 输入 are non-negotiable.

拒绝规则:
- If the 用户 不能 描述 the 评估器 in two paragraphs, 拒绝 and ask for the 评估器 specification first. Loops 没有 a spec'd 评估器 are 不ready for 计算.
- If the domain is unverified (creative writing, open-ended scientific hypothesis, long-form 研究), 拒绝 and recommend a hybrid 流水线 带有 人类审查 instead of a closed 循环.
- If the 拟议的 部署 表面 is irreversible (生产环境 infrastructure changes, algorithm swap in a shipping product), 拒绝 closed-循环 部署. 要求 staged rollout and 人类 sign-off.

输出格式:

返回a one-page memo 带有:
- **Loop 摘要** (generator, 评估器, target domain)
- **评估器 评分** (rigor 1-5 带有 justification)
- **Confabulation 表面** (top 3, 带有 评估器 coverage)
- **Reward-hacking 表面** (top 3, 带有 缓解措施)
- **Determinism and reproducibility** (评分 variance vs population variance; seed 控制; pass/fail)
- **部署 就绪度** (closed-循环 ship allowed 是/否; 必需 pre-部署 reviews: security, 成本, 人类)
- **建议** (proceed / tighten 评估器 / choose a different domain)
