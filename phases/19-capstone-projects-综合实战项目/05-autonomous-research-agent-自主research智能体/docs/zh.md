# Capstone 05 — 自主研究智能体（AI-Scientist 类）

> Sakana 的 AI-Scientist-v2 发表了完整论文。Agent Laboratory 跑了实验。Allen AI 分享了 traces。2026 年的形态是：对实验做 plan-execute-verify tree search，带预算成本、沙箱代码执行、带视觉反馈的 LaTeX writer，以及自动化 NeurIPS 风格 reviewer ensemble。本综合实战是构建一个系统，让它在每篇论文 30 美元内端到端运行，并通过 Sakana 记录的 sandbox-escape red team。

**Type:** Capstone
**Languages:** Python（agent + sandbox）、LaTeX（output）
**Prerequisites:** Phase 2（ML）、Phase 3（deep learning）、Phase 7（transformers）、Phase 10（LLMs from scratch）、Phase 14（agents）、Phase 15（autonomous）、Phase 16（multi-agent）、Phase 18（safety）
**Phases exercised:** P0 · P2 · P3 · P7 · P10 · P14 · P15 · P16 · P18
**Time:** 40 小时

## 问题

自主研究智能体在 2026 年跨过了门槛。Sakana AI 的 AI-Scientist-v2 登上 Nature，生成论文通过了 workshop peer review。ShinkaEvolve（ICLR 2026）把这条线扩展到 evolving hypotheses。AMD 的 Agent Laboratory 交付了可复现 traces。这些智能体不是魔法，它们是在候选实验树上运行的 plan-execute-verify loop，带成本上限、seed-bound sandboxes 和自动评审。工艺在循环、预算和安全叙事里。

你会通过在狭窄领域的一个 seed idea 上实现它来学习循环，例如 100M 参数 transformer 上的 attention-sparsity ablations。价值不在第一次运行就发现新东西。价值在基础设施：tree-search、experiment sandbox、writer-reviewer loop、red-team report。Sakana 团队记录了 sandbox-escape failures；你的智能体必须通过同一套 red team。

## 概念

智能体是 best-first tree search。节点是实验规格：(hypothesis, config, code, expected outcome)。expand 步骤提出带小编辑的 children（换 optimizer、改变 batch size、ablate 一个组件）。每个 child 都在全新 sandbox 中运行，带硬资源上限。结果反馈进 scoring function，按 (novelty × quality × remaining budget) 对节点排序。树增长到预算耗尽，然后把最佳分支写成论文。

Writer 是多模态的。它生成 LaTeX draft、编译、渲染 figures，并把渲染后的 PDF 送回 Claude Opus 4.7 vision mode，让它 critique layout、figure legibility 和 claim-evidence alignment。由五个 LLM judges 组成的 reviewer ensemble 输出 NeurIPS 风格 scores（novelty、rigor、clarity、reproducibility、impact）；如果平均值低于阈值，论文带 critique 返回 writer。

安全是关键承重件。每个实验都在没有网络出口、墙钟时间受限、资源限制固定的 E2B 或 Daytona sandbox 中运行。智能体的代码生成步骤经过 policy layer，阻止能逃逸 sandbox 的 syscalls。Red-team report 复现 Sakana 记录的攻击面（fork bombs、filesystem escapes、LLM-written network calls）。

## 架构

```
seed idea + domain
      |
      v
  literature search (Semantic Scholar + OpenAlex + FAISS cache)
      |
      v
  LangGraph plan-execute-verify tree
      |
      v
  +--- expand node ----+      per-node sandbox
  |                    |      (E2B / Daytona)
  v                    v      resource caps
  child_1           child_k   no network egress
  |                    |      deterministic seeds
  v                    v
  run experiment       run experiment
  |                    |
  v                    v
  score nodes by (novelty, quality, budget)
      |
      v
  best branch -> LaTeX writer
      |
      v
  compile + vision critique (Opus 4.7 vision)
      |
      v
  reviewer ensemble (5 LLM judges, NeurIPS rubric)
      |
      v
  paper.pdf + review.md + trace.json
```

## 技术栈

- 编排：带 checkpointing 和 human-approval gates 的 LangGraph
- Tree search：在实验节点上自定义 best-first（Sakana v2 的 AB-MCTS-style）
- Sandbox：每个实验一个 E2B，Docker-in-Docker fallback；通过 cgroups 做 resource caps
- 文献：Semantic Scholar Graph API + OpenAlex + 本地 abstracts FAISS cache
- Writer：LaTeX template + Claude Opus 4.7（vision mode），用于 figure critique 和 layout
- Reviewer：5 个 judges ensemble（Opus 4.7、GPT-5.4、Gemini 3 Pro、DeepSeek R1、Qwen3-Max），加权聚合
- 实验框架：PyTorch 2.5 用于物理实验，W&B 用于 logging
- 可观测性：Langfuse 记录 agent traces，每篇论文 30 美元硬预算

## 构建它

1. **Seed 和 domain scoping。** 选一个 seed idea（例如 “investigate sparsity patterns in attention maps of sub-1B transformers”）。定义搜索空间：models、datasets、compute budget。

2. **文献阶段。** 查询 Semantic Scholar + OpenAlex，拿到 50 篇最相关的高引用论文；本地缓存 abstracts；生成 1 页 domain digest。

3. **树脚手架。** 用 seed hypothesis 初始化 root。实现 `expand(node) -> children`，用小编辑 proposals（每个 child 改一个 config）。实现 `score(node)`，作为加权 novelty × quality × budget 项。

4. **沙箱包装。** 每个实验运行 `docker run --network=none --memory=8g --cpus=2 --pids-limit=256 --read-only`（或等价 E2B policy）。Seeds 写入 sandbox；outputs 以只读方式挂载回外部。

5. **Plan-execute-verify 循环。** `plan` 提出 children。`execute` 运行 sandbox，捕获 logs 和 metrics。`verify` 对 metrics 运行 unit checks（loss 是否下降？ablation 是否隔离了效果？）。失败节点在树上存储 failure reason。

6. **Writer。** 预算结束后选择最佳分支。用 matplotlib 渲染 figures。用 Claude Opus 4.7 生成 LaTeX draft，并把 branch trace 放进上下文。编译。把编译后的 PDF 送回 Opus 4.7 vision 做 critique。迭代。

7. **Reviewer ensemble。** 五个 judges 用 NeurIPS 风格 rubrics 按 (novelty, rigor, clarity, reproducibility, impact) 给 draft 打分。如果 mean < 4.0/5，带 critique 返回 writer。3 次 rewrites 后 hard stop。

8. **红队。** 构建或集成一组针对 sandbox 的 adversarial tasks：fork bombs、network exfiltration attempts、filesystem escapes、LLM-written shell metacharacters。确认全部被阻止。写出 findings。

9. **可复现性。** 每篇论文都带 tree-search trace JSON、seeds、W&B run links、sandbox configs，以及能端到端复现的 README。

## 使用它

```
$ ai-scientist run --seed "attention sparsity in sub-1B transformers" --budget 30
[lit]    50 papers, digest in 12s
[tree]   expanded 8 nodes, budget 12/30
[exec]   node #3 sparsity=top-8, loss=2.83 (best so far)
[exec]   node #6 sparsity=top-4, loss=3.12 (worse)
[exec]   ...
[tree]   chose branch rooted at node #3 (novelty 0.62, quality 0.81)
[write]  LaTeX draft v1 complete
[vision] critique: figure 2 legend too small, claim-evidence ok
[write]  draft v2 after 3 edits
[review] mean 4.2/5 (novelty 3.9, rigor 4.3, clarity 4.1, repro 4.5, impact 4.2)
[done]   paper.pdf + review.md + trace.json     $28.40 spent
```

## 交付它

`outputs/skill-ai-scientist.md` 是交付物。给定 seed idea + domain + 30 美元预算，它会运行完整流水线，并输出一篇可评审论文和可复现 bundle。

| Weight | Criterion | How it is measured |
|:-:|---|---|
| 25 | 论文质量 | 与已发表 workshop papers 做盲评 rubric review |
| 20 | 实验严谨性 | Baselines、seeds、ablations；每个主张由结果表中的 cell 支撑 |
| 20 | 成本和计算纪律 | 强制 $30/paper 上限，由 Langfuse trace |
| 20 | 安全 | 通过 sandbox red team；network policy 和 kill-switch 已验证 |
| 15 | 可复现性 | 一条命令 rerun，用相同 seeds 复现论文 |
| **100** | | |

## 练习

1. 在同一领域的三个不同 seed ideas 上运行流水线。比较 tree-search 的哪些部分重叠。识别重复浪费的 compute。

2. 对预计超过 5 美元的节点，在实验执行前增加 human-in-the-loop gate。度量总成本下降多少。

3. 把 reviewer ensemble 替换为 single judge。在 known-bad papers 的 held-out set 上度量 false-accept rate。

4. 引入 network-exfiltration red team test：agent 写代码尝试 `curl` 外部地址。确认 `--network=none` policy 阻止它。记录尝试。

5. 将你的 tree-search 与 flat random baseline 对比（相同预算，没有 expansion strategy）。报告 novelty × quality gain。

## 关键术语

| Term | What people say | What it actually means |
|------|-----------------|------------------------|
| Tree search | “AB-MCTS-style expansion” | 在实验节点上按 novelty×quality×budget score 做 best-first exploration |
| Sandbox | “Experiment isolation” | 无网络、CPU/内存受限、seeds 固定、inputs 只读的容器 |
| Vision critique | “Render-then-read” | 把论文编译成 PDF，再送回 VLM 做 layout 和 claim-evidence critique |
| Reviewer ensemble | “Automated peer review” | 多个 LLM judges 用 NeurIPS rubric 给论文打分；加权聚合 gate 流水线 |
| Novelty score | “Is this new?” | 惩罚与 50-paper literature cache 过近的 heuristic |
| Cost ceiling | “$ budget” | 每篇论文总花费硬上限；Langfuse counters + pre-run estimates |
| Red team | “Sandbox-escape audit” | 如果 policy 错误就会逃逸 sandbox 的 adversarial tasks |

## 延伸阅读

- [Sakana AI-Scientist-v2 repository](https://github.com/SakanaAI/AI-Scientist-v2) — 生产研究智能体参考
- [Sakana AI-Scientist-v1 paper (arXiv:2408.06292)](https://arxiv.org/abs/2408.06292) — 原始方法论文
- [ShinkaEvolve (Sakana ICLR 2026)](https://sakana.ai) — 进化式扩展
- [Agent Laboratory (AMD)](https://github.com/SamuelSchmidgall/AgentLaboratory) — 多角色 research-lab framework
- [LangGraph documentation](https://langchain-ai.github.io/langgraph/) — 参考 orchestration layer
- [Semantic Scholar Graph API](https://api.semanticscholar.org/) — 文献搜索
- [E2B sandboxes](https://e2b.dev) — 实验隔离参考
- [NeurIPS reviewer guidelines](https://neurips.cc/Conferences/2026/Reviewer-Guidelines) — reviewer ensemble 编码的 rubric
