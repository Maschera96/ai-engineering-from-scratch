# 自主编程智能体格局（2026）

> SWE-bench Verified went 来自 4% to 80.9% in under three years. Same Claude Sonnet 4.5 scored 43.2% on SWE-智能体 v1 and 59.8% on Cline 自主 — the scaffolding around the 模型 now matters as much as the 模型 itself. OpenHands (formerly OpenDevin) is the most active MIT-licensed platform and its CodeAct 循环 executes Python actions 直接 in a 沙箱 instead of JSON 工具 calls. The headline numbers hide a methodological issue: 161 of 500 SWE-bench Verified 任务 要求 只 a 1–2 行 change, and SWE-bench Pro (10+ 行 任务) sits at 23–59% for the same 前沿模型.

**Type:** Learn
**Languages:** Python (stdlib, CodeAct vs JSON 工具-call comparison)
**Prerequisites:** Phase 14 · 07 (工具 use), Phase 15 · 01 (Long-horizon agents)
**Time:** ~45 minutes

## 问题

"Which coding 智能体 is best" is the wrong question. The right question is: on a 任务 分布 that matches my work, 带有 the scaffolding I will 运行 in 生产环境, 什么 end-to-end 可靠性 do I get?

在2022 and 2026 the field learned that scaffolding — the retrieval 层, the planner, the 沙箱, the edit-验证 循环, the 反馈 format — is load-bearing. Claude Sonnet 4.5 on SWE-智能体 v1 scored 43.2% on SWE-bench Verified; the same 模型 内部 Cline's 自主 scaffold scored 59.8%. 16.6 absolute points of difference, same 权重. The base 模型 is a component; the 循环 is the product.

这个companion problem is that 基准 saturation hides regressions. SWE-bench Verified is close to saturated, and the easy-任务 tail (161 of 500 任务 requiring ≤2 lines) pulls top 分数 up. Real-world 质量 is better measured on 分布 like SWE-bench Pro (10+ 行 changes), 在哪里 the same leaders still sit at 23–59%.

## 概念

### SWE-bench, 一段话

SWE-bench (Jimenez et al.) takes real GitHub issues 带有 ground-truth patches and asks an 智能体 to 生成 a patch that makes the test suite pass. SWE-bench Verified (OpenAI, 2024) is a 人类-curated 500-任务 subset 带有 the 含糊 and broken 任务 removed. SWE-bench Pro is the harder successor — 任务 requiring 10+ lines of change, 在哪里 current frontier agents sit at 23–59%.

### 内容 the 2022 → 2026 curve actually shows

- **2022**: 研究 models at ~4% on raw SWE-bench.
- **2024**: GPT-4 + Devin-style scaffolding at ~14%; SWE-智能体 at ~12%.
- **2025**: Claude 3.5/3.7 Sonnet 内部 Aider and SWE-智能体 push 进入 the 40–55% range.
- **2026**: Claude Sonnet 4.5 and frontier competitors at 70–80%+ on SWE-bench Verified. Epoch AI's leaderboard tracks this live.

这个slope came 来自 three compounding 来源: better base models, better scaffolding (CodeAct, reflection, verifier 循环), and better 基准s (Verified removing noise).

### CodeAct vs JSON 工具 calls

OpenHands (所有-Hands-AI, arXiv:2407.16741, formerly OpenDevin) took a 具体 architectural bet: instead of the 模型 emitting JSON 工具 calls that a host decodes and executes, the 模型 emits Python code and a Jupyter-style kernel runs it in a 沙箱. The 智能体 can 循环 over 文件, chain 工具, and catch its own exceptions 内部 one action.

这个trade-off:

- **JSON 工具 calls**: 每个 action is one turn; easy to 审计; limited compositionality; safe by 默认值 because each call goes through an 明确 validator.
- **CodeAct**: one action can be a whole program; compositional; 要求 a hardened 沙箱 (OpenHands uses Docker 隔离); 失败 modes include anything the 沙箱 runtime allows.

两者 architectures are in 生产环境. CodeAct is dominant in open platforms (OpenHands, smolagents). JSON 工具 calls remain dominant in managed services (Anthropic Managed Agents, OpenAI Assistants) 在哪里 the provider 控制 the executor.

### Scaffolds in the 2026 landscape

| Scaffold | License | Execution 模型 | Notable property |
|---|---|---|---|
| OpenHands (OpenDevin) | MIT | CodeAct in Docker | Most active open platform; event-stream replayable |
| SWE-智能体 | MIT | Agent-Computer Interface (ACI) | First end-to-end SWE-bench scaffold |
| Aider | Apache-2 | edit-via-diff in local repo | Minimal scaffold, strong 回归 stability |
| Cline | Apache-2 | VS Code 智能体 带有 工具 政策 | Highest-scoring open scaffold on Sonnet 4.5 |
| Devin (Cognition) | Proprietary | Managed VM + planner | First "AI software engineer" product 类别 |
| Claude Code | Proprietary | Permission modes + routines | Lesson 10 covers the 智能体 循环 in detail |

### 原因 scaffolding dominates

一个coding 运行 is a long-horizon 轨迹 (Lesson 1). Reliability compounds across steps. Three places 在哪里 scaffolding buys points:

1. **Retrieval**: finding the right 文件 to 阅读 is the silent bottleneck. SWE-智能体's ACI, OpenHands' 文件-index, and Aider's repo-映射 所有 攻击 this.
2. **Verifier 循环**: running tests, reading stack traces, and re-attempting is a 10+ point delta on SWE-bench.
3. **失败 containment**: a 沙箱 that rolls back on error prevents compounding damage. The same 模型 带有 and 没有 a verifier 循环 looks like two different products.

### 基准 saturation and the real 分布

这个OpenHands authors and Epoch AI 两者 标出 that SWE-bench Verified has an easy tail: 161 of 500 任务 need 只 1–2 lines of change. High 分数 are driven partly by this tail. SWE-bench Pro restricts to 10+ 行 changes and returns 分数 in the 23–59% range even for frontier systems. Your 生产环境 分布 is almost certainly closer to Pro than to Verified.

Implication for choosing an 智能体: 运行 a Pro-like subset of your own bug backlog. The 评分 that matters is the 评分 on 任务 representative of 什么 you ship.

## 使用它

`code/main.py` compares two toy 智能体 scaffolds on a fixed mini-任务 分布:

1. A **JSON 工具-call** scaffold that takes one action per turn.
2. A **CodeAct** scaffold that can emit a small Python snippet per action.

两者 use a stub "模型" (deterministic rules) so the comparison isolates the scaffold 来自 模型 质量. The 输出 shows the CodeAct scaffold solves more 任务 in fewer turns at the 成本 of a larger per-action blast radius.

## 交付它

`outputs/skill-scaffold-audit.md` helps you 审计 a 拟议的 coding-智能体 scaffold 之前 adoption: retrieval 质量, verifier presence, 沙箱 隔离, and 基准-to-分布 fit.

## 练习

1. 运行 `code/main.py`. 方式 many turns does each scaffold take on the same 任务 设置? 内容 is the per-action blast radius of each?

2. 阅读 the OpenHands 论文 (arXiv:2407.16741). The 论文 argues CodeAct beats JSON 工具 calls on complex 任务. 识别 one 失败 mode the 论文 acknowledges and 编写 一句话 on when that mode would dominate in 生产环境.

3. Pick one 任务 来自 your bug backlog that would 要求 10+ lines of change across two 文件. 估计 the end-to-end success probability for a 前沿模型 under (a) JSON 工具 calls and (b) CodeAct. Justify the 缺口.

4. SWE-bench Verified has 161 单一-文件, 1–2 行 任务. Construct a 评分 that excludes them. 方式 does the leaderboard shuffle?

5. 阅读 "Introducing SWE-bench Verified" (OpenAI). 解释 the 具体 方法 used to remove 含糊 任务, and 命名 one 类别 the curation would miss.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| SWE-bench | "Coding 基准" | Real GitHub issues 带有 ground-truth patches and test suites |
| SWE-bench Verified | "Cleaned subset" | 500 人类-curated 任务, easier-tail 存在 |
| SWE-bench Pro | "Harder subset" | 10+ 行 changes; frontier sits at 23–59% |
| CodeAct | "Code-as-action" | Agent emits Python; Jupyter-style kernel executes in 沙箱 |
| JSON 工具 call | "Function calling" | 每个action is a structured JSON payload validated 之前 execution |
| Scaffold | "Agent framework" | Retrieval + planner + executor + verifier 循环 around the base 模型 |
| ACI (Agent-Computer Interface) | "SWE-智能体's format" | Command 设置 designed for LLM ergonomics, 不human shells |
| Verifier 循环 | "Test-and-retry" | 运行 tests, 阅读 输出, revise patch; biggest non-模型 可靠性 gain |

## 延伸阅读

- [Jimenez et al. — SWE-bench](https://www.swebench.com/) — the original 基准 and 方法.
- [OpenAI — Introducing SWE-bench Verified](https://openai.com/index/introducing-swe-bench-verified/) — 如何 the curated subset was built.
- [Wang et al. — OpenHands: An Open Platform for AI Software Developers](https://arxiv.org/abs/2407.16741) — CodeAct architecture and event-stream 设计.
- [Epoch AI — SWE-bench leaderboard](https://epoch.ai/benchmarks) — live-tracked 分数.
- [Anthropic — Measuring agent autonomy](https://www.anthropic.com/research/measuring-agent-autonomy) — long-horizon coding-智能体 可靠性 framing.
