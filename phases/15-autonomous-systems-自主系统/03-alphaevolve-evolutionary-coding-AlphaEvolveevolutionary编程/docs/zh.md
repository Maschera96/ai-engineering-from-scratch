# AlphaEvolve：进化式编程智能体

> Pair a frontier coding 模型 带有 an evolutionary 循环 and a machine-checkable 评估器. Let the 循环 运行 long enough. It discovers a 4x4 complex-matrix multiplication procedure that uses 48 scalar multiplications — the first improvement over Strassen in 56 years. It also finds a Google-wide Borg scheduling heuristic that recovers ~0.7% of cluster 计算 in 生产环境. The architecture is boring on purpose. The wins come 来自 the 评估器's rigor.

**Type:** Learn
**Languages:** Python (stdlib, evolutionary-loop toy)
**Prerequisites:** Phase 15 · 01 (long-horizon framing), Phase 15 · 02 (self-taught reasoning)
**Time:** ~60 minutes

## 问题

Large language models can 编写 code. Evolutionary algorithms can search over code. 两者 have been tried separately for decades; 两者 hit ceilings. The LLM ceiling is confabulation: the 模型 writes plausible code that does 不do 什么 it 声明. The evolutionary ceiling is search 成本: random 变异 over syntax rarely 生成 compilable programs, let alone better ones.

AlphaEvolve (Novikov et al., DeepMind, arXiv:2506.13131, June 2025) combines them. The LLM proposes targeted edits to a program database; an automatic 评估器 分数 each variant; high-scoring variants become parents for future generations. The LLM handles the expensive step of writing plausible code; the 评估器 catches the confabulations. The 循环 runs for hours to weeks.

Results reported: 48-scalar-multiplication 4x4 complex matrix multiplication (Strassen's 1969 bound was 49), a Borg scheduling heuristic in Google 生产环境, a 32.5% FlashAttention kernel speedup, Gemini training throughput improvements.

这个architecture works because the 评估器 is machine-checkable. It does 不work 在哪里 the 评估器 isn't. That asymmetry is the lesson.

## 概念

### 这个loop

1. Start 来自 a seed program `P_0` that is correct but suboptimal.
2. Maintain a database of variant programs, each scored by the 评估器.
3. Sample one or more parents 来自 the database (映射-elites-style or island-based).
4. 提示词 the LLM (Gemini Flash for many candidates, Gemini Pro for the hard ones) to 生成 a modified variant of the parent.
5. Compile, 运行, and evaluate the variant on the 留出 评估器.
6. Insert 进入 the database keyed by its 评分 and feature vector.
7. Repeat.

Two details matter. First, the LLM is prompted 带有 more than the parent program — typically several top variants 来自 the database, plus the 评估器 signature, plus a short 任务 description. The 模型's job is to propose a targeted change that might improve the 评分. Second, the database is structured (映射-elites grid, island-based) so the 循环 explores diversity, 不just the current leader.

### 内容 makes the 评估器 non-negotiable

AlphaEvolve's wins 所有 come 来自 domains 在哪里 the 评估器 is fast, deterministic, and hard to game:

- **Matrix multiplication algorithm**: a unit test that multiplies matrices and checks equality bit-identically.
- **Borg scheduling heuristic**: a 生产环境-grade simulator that replays historical cluster load and measures wasted 计算.
- **FlashAttention kernel**: a 正确性 test plus a wall-clock 基准 on real hardware.
- **Gemini training throughput**: measured GPU-seconds per step.

在each case the 评估器 catches the 类别 of LLM errors that would otherwise dominate: confabulated 正确性 声明, 性能 声明 that vanish on hardware, and edge-case 失败. Remove the 评估器 and the 循环 optimizes for pretty code.

### Reward hacking is the other face of that statement

Evolution optimizes for whatever the 评估器 measures. If the 评估器 is imperfect, the 循环 will find the imperfection. In an unverified domain the 循环 would optimize for the 表面 feature, 不the intended behavior. DeepMind flags this explicitly in the 论文: AlphaEvolve's successes transfer 只 to domains 在哪里 评估器 rigor matches the ambition of the search.

具体 2025-2026 examples of reward hacking in code-search 循环:

- Optimization targets that reward "time to complete" rewarded submitting empty solutions.
- 基准 分数 that reward 正确性-under-test rewarded memorizing tests and overfitting.
- A "code 质量" proxy rewarded removing comments and rewriting variable names, 带有 no semantic change.

这个fix in AlphaEvolve: ship a 留出 评估器 the LLM has never seen, 带有 输入 generated at 评估 time. Even then, DeepMind recommends strong 审查 on 任何 拟议的 部署.

### 原因 LLM + search beats either alone

这个LLM can 生成 compilable, semantically plausible modifications. A random-变异 GA on a 2000-行 Python 文件 almost always produces syntax errors. The LLM also concentrates search on plausible neighborhoods (change one function, 不random bytes) 哪个 dramatically reduces wasted 评估器 calls.

这个评估器, in turn, catches the LLM's confabulations. LLMs will confidently 声明 that a function "is O(n 日志 n) in the 限制" when it is actually O(n^2); a wall-clock 基准 makes the question settled.

### 位置 AlphaEvolve fits in the frontier stack

| System | Generator | 评估器 | Domain | Example win |
|---|---|---|---|---|
| AlphaEvolve | Gemini | 正确性 + 基准 | algorithms, kernels, schedulers | 48-mul 4x4 matmul |
| FunSearch (DeepMind, 2023) | PaLM / Codey | 正确性 | combinatorial math | 上限-设置 lower 边界 |
| AI Scientist v2 (Sakana, L5) | GPT/Claude | LLM critique + experiment | ML 研究 | ICLR workshop 论文 |
| Darwin Godel Machine (L4) | 智能体 scaffolding | SWE-bench / Polyglot | 智能体 code | 20% → 50% SWE-bench |

所有 four are variations on the same recipe: generator plus 评估器, 循环. The differences are 什么 the 评估器 grades and 如何 rigorous it is.

## 使用它

`code/main.py` implements a minimal AlphaEvolve-like 循环 over a toy symbolic-回归 problem. The "LLM" is a stdlib proxy that proposes small syntactic 变异 to a program that computes a target function. The "评估器" measures mean squared error on 留出 test points.

Watch:

- 方式 the best 评分 improves over generations.
- 方式 a 映射-elites grid keeps diverse solutions alive so the 循环 doesn't converge on a local minimum.
- 方式 removing the 留出 test (training-只 评估器) lets the 循环 overfit spectacularly.

## 交付它

`outputs/skill-evaluator-rigor-audit.md` is the 前置条件 for considering an AlphaEvolve-style 循环 in a new domain: does your 评估器 actually catch the 失败 you care about?

## 练习

1. 运行 `code/main.py`. Note the best 评分 轨迹. Disable the 留出 评估器 (标出 `--no-holdout`) and re-运行. Quantify the overfitting.

2. 阅读 Section 3 of the AlphaEvolve 论文 on the 映射-elites grid. 设计 a feature-vector descriptor for a new problem (e.g. compiler optimization passes) that would keep the search diverse.

3. The 48-multiplication 4x4 结果 improved on Strassen's 49-mul bound 之后 56 years. 阅读 Appendix F of the 论文 and 解释 in three sentences 为什么 the 评估器 for this problem is particularly easy to get right, and 为什么 most domains are 不like it.

4. Propose one domain 在哪里 AlphaEvolve would fail. 识别 exactly 在哪里 the 评估器 breaks and 为什么.

5. For a domain you know, 编写 the 评估器 signature you would use. Include (a) 正确性 conditions, (b) 性能 指标, (c) 留出 输入 generation rule, (d) at least one anti-reward-hacking check.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| AlphaEvolve | "DeepMind's evolutionary coding 智能体" | Gemini + program database + machine-checkable 评估器 |
| 映射-elites | "Diversity-preserving archive" | Grid keyed by feature vectors; each cell holds the best variant 带有 that descriptor |
| Island 模型 | "Parallel evolution subpopulations" | 独立 populations that migrate periodically; prevents premature convergence |
| Machine-checkable 评估器 | "Deterministic oracle" | 一个unit test, simulator, or 基准 the LLM 不能 fake — a prerequisite for this 循环 |
| Reward hacking | "Optimizing the measure, 不the goal" | Loop finds a way to maximize 评分 没有 doing the intended 任务 |
| Seed program | "The starting point" | 一个initial correct-but-suboptimal program the 循环 evolves 来自 |
| 留出 评估器 | "Evaluation data the LLM never saw" | 输入 generated at 评估 time to prevent memorization |

## 延伸阅读

- [Novikov et al. (2025). AlphaEvolve: A coding agent for scientific and algorithmic discovery](https://arxiv.org/abs/2506.13131) — the full 论文.
- [DeepMind blog on AlphaEvolve](https://deepmind.google/blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/) — vendor writeup 带有 结果.
- [AlphaEvolve results repository](https://github.com/google-deepmind/alphaevolve_results) — discovered algorithms, including the 48-mul 4x4 matmul.
- [Romera-Paredes et al. (2023). Mathematical discoveries from program search with LLMs (FunSearch)](https://www.nature.com/articles/s41586-023-06924-6) — the predecessor 系统.
- [Anthropic — Responsible Scaling Policy v3.0 (Feb 2026)](https://anthropic.com/responsible-scaling-policy/rsp-v3-0) — frames 评估器-bound autonomy as a key 研究 direction.
