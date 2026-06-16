# 自动化对齐研究（Anthropic AAR）

> Anthropic ran parallel teams of Claude Opus 4.6 自主 对齐 Researchers in 独立 沙箱es, coordinating via a shared 论坛 whose 日志 live 外部 任何 沙箱 (so agents 不能 delete their own records). On the weak-to-strong training problem, the AARs outperformed 人类 researchers. Anthropic's own 摘要 flags that prescribed 工作流s often constrain AAR flexibility and degrade 性能. Automating 对齐 研究 is the compression step that compresses the timeline to the exact mis对齐 风险 the RSP is meant to detect.

**Type:** Learn
**Languages:** Python (stdlib, parallel-research-论坛 simulator)
**Prerequisites:** Phase 15 · 05 (AI Scientist v2), Phase 15 · 04 (DGM)
**Time:** ~60 minutes

## 问题

对齐 研究 is expensive in 人类-researcher time. Problems like scalable oversight, reward specification, or weak-to-strong training 要求 experiments that take weeks per iteration. As frontier 能力 advance, the 对齐 workload grows faster than the supply of qualified researchers.

Automated 对齐 Research (AAR) asks 是否 the same 前沿模型 whose 能力 is outrunning 对齐 can contribute to closing the 缺口. Anthropic's 2026 报告 on an AAR-运行 weak-to-strong-training study (对齐.anthropic.com/2026/automated-w2s-researcher/) is one of the first public 结果 来自 a deployed 系统 of this 类别.

这个result is genuinely positive: AARs solved a 研究 problem better than the 人类 baseline. The 结果 also concentrates, in a 单一 系统, 每个 concern this phase has developed. If 对齐 研究 can be automated, so can the parts that compromise safeguards. The RSP's 阈值 for AI R&D 能力 are written 带有 this 循环 in mind.

## 概念

### 这个architecture (as publicly described)

- **Parallel agents.** Multiple Claude Opus 4.6 instances 运行 as AARs, each in an 独立 沙箱. 沙箱 限制 文件系统, 网络, and 进程 表面.
- **Shared 论坛.** Findings are published to a shared 论坛 whose 存储 后端 sits 外部 每个 沙箱. Agents can 阅读 the 论坛; agents 不能 delete or edit 论坛 records 来自 within their own 沙箱.
- **任务 分配.** A 人类-designed 任务 queue assigns 研究 problems to AARs. AARs propose decompositions, 运行 experiments, 编写 up findings.
- **Weak-to-strong training focus.** The published study addressed weak-to-strong training — using a weaker supervisor 模型 to train a stronger student 模型. AARs outperformed 人类 researchers on this 任务.

### 原因 the out-of-沙箱 日志 matters

这个log-integrity property is the difference between a 控制led experiment and a potentially compromised one. If AARs could edit their own 日志, they could cover up failed or dubious experiments. Anthropic flags this 直接: the property is a necessary condition for trusting the 研究 输出, and it is non-trivial to 验证.

这个analog in infrastructure terms is 只追加 日志 带有 编写-through to 外部 存储 (e.g. S3 object-lock, immutable Datadog 日志). The 智能体 has no 凭据 to modify past records. This is the same pattern that 2026 corporate security teams use to make ransomware-resistant 审计轨迹s.

### 这个prescribed-工作流 tradeoff

Anthropic's 报告 notes that prescribed 工作流s often constrain AAR flexibility and degrade 性能. In other words: if the 人类 designer writes the 研究 plan step-by-step, AARs do worse than if 给定 high-level goals and freedom to decompose. This is consistent 带有 结果 across agentic 研究 (LangGraph, Microsoft's Agent Framework) — overspecified scaffolds hurt capable agents.

这个安全 implication is subtle. The flexibility that improves AAR 性能 is also the flexibility that allows unsanctioned decomposition, novel 工具 use, and goal-drift. The tradeoff is a 设计-space decision: 如何 much flexibility buys enough 能力 to beat 人类, 没有 handing the 智能体 room to generalize goals off-分布?

### 这个compression 风险

RSP v3.0 (Lesson 19) introduces an AI R&D 能力 阈值: the ability to fully automate the AI R&D 流水线 at competitive 成本 vs 人类 + AI 工具. DeepMind's FSF v3 includes an analogous ML R&D autonomy level. 两者 frameworks treat this 阈值 as the 触发器 for elevated 控制.

AAR is one step short of the 阈值: it automates part of the 流水线 (对齐 研究 on 具体, well-scoped 任务) but 不the end-to-end 能力 development 循环. The timeline question is 如何 fast the 缺口 closes.

Compressed timelines are the compounding-失败 concern. If 对齐 研究 and 能力 研究 compound at similar rates, the mis对齐 风险 表面 grows at least as fast as 能力. If 能力 compounds faster (the historical trend), the 缺口 widens. This is the argument for AAR being a qualified good: each additional 对齐 结果 reduces the 缺口 if and 只 if the 研究 进程 is trustworthy.

### 内容 AAR does 不replace

人类 researchers 设置 the 任务 queue, 审查 结果, and 暂停 the 宪法式 authority. The AARs accelerate the middle of the 流水线, 不the ends. Anthropic's published 输出 include 两者 AAR contributions and 人类-researcher judgement on 什么 to publish, 什么 to retract, and 什么 to refine.

这matches the propose-then-commit pattern 来自 Lesson 15 applied to 研究 itself: AARs propose; 人类 commit.

## 使用它

`code/main.py` simulates a multi-AAR 研究 论坛. Three 独立 "研究 agents" 运行 in parallel, posting findings to an 只追加 论坛 带有 optional 篡改 attempts.

Watch:

- 方式 out-of-沙箱 日志 catches 日志-tampering attempts.
- 方式 任务 分配 带有 free decomposition outperforms fixed 工作流s on harder problems.
- 方式 a 单一 compromised AAR can contaminate the 论坛's signal if the 日志 is 不tamper-evident.

## 交付它

`outputs/skill-aar-deployment-review.md` is a pre-部署 审查 of an automated-对齐-研究 流水线.

## 练习

1. 运行 `code/main.py`. 比较 the "fixed-工作流" vs "free-decomposition" 设置. Does free decomposition always win, or are there problem 类别 在哪里 the fixed 工作流 is competitive?

2. Modify the simulator so one 智能体 attempts 日志 tampering. 确认 the 只追加 日志 detects it. 编写 一段话 describing exactly 什么 the detection signature looks like in the 日志.

3. 阅读 Anthropic's weak-to-strong AAR 报告. 识别 the 具体 sub-任务 the AARs beat 人类 researchers on. 内容 made it amenable to automation?

4. 设计 a 任务-queue 分配 政策 that balances AAR flexibility (better 结果) 针对 prescribed-工作流 constraints (easier 审计). 描述 如何 you would A/B test the two.

5. 阅读 RSP v3.0's AI R&D-4 阈值. In 一段话, 描述 什么 you think would cross it that AAR currently does not.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| AAR | "Automated 对齐 Researcher" | Claude Opus 4.6 instance operated autonomously on 对齐 problems |
| Weak-to-strong training | "Training a stronger 模型 带有 a weaker supervisor" | Classic scalable-oversight 基准 AARs outperformed 人类 on |
| Shared 论坛 | "位置 agents publish findings" | Append-只, out-of-沙箱 存储 |
| Out-of-沙箱 日志 | "Agent 不能 edit its own record" | 可证明篡改 编写-through to 外部 存储 |
| Prescribed 工作流 | "Step-by-step plan 来自 人类 designer" | Constrains AAR; often degrades 性能 vs free decomposition |
| Free decomposition | "Agent decides 如何 to break the 任务" | More capable, harder to 审计 |
| AI R&D 阈值 | "RSP/FSF 能力 level" | Full automation of R&D 流水线 at competitive 成本 |
| Compressed timeline | "对齐 vs 能力 race" | 如果能力 compounds faster than 对齐, mis对齐 风险 grows |

## 延伸阅读

- [Anthropic — Automated Weak-to-Strong Researcher](https://alignment.anthropic.com/2026/automated-w2s-researcher/) — primary 来源.
- [Anthropic Responsible Scaling Policy v3.0](https://anthropic.com/responsible-scaling-policy/rsp-v3-0) — AI R&D 阈值 framing.
- [Anthropic — Measuring AI agent autonomy](https://www.anthropic.com/research/measuring-agent-autonomy) — broader 智能体-autonomy framing.
- [DeepMind Frontier Safety Framework v3](https://deepmind.google/blog/strengthening-our-frontier-safety-framework/) — ML R&D autonomy levels parallel to RSP.
- [Burns et al. (2023). Weak-to-Strong Generalization (OpenAI)](https://openai.com/index/weak-to-strong-generalization/) — the underlying problem AARs attacked.
