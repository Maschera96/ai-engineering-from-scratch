# AI Scientist v2：研讨会级自主研究

> Sakana's AI Scientist v2 (Yamada et al., arXiv:2504.08066) runs the full 研究 循环: hypothesis, code, experiments, figures, writeup, submission. It is the first 系统 to have a generated 论文 pass peer 审查 at an ICLR 2025 workshop. 独立 评估 (Beel et al.) found 42% of experiments failed 来自 coding errors and literature 审查 frequently mislabeled established concepts as novel. Sakana's own docs warn that the codebase executes LLM-written code and recommend Docker 隔离. 两者 halves of that picture are the point.

**Type:** Learn
**Languages:** Python (stdlib, research-loop 说明-machine toy)
**Prerequisites:** Phase 15 · 03 (AlphaEvolve), Phase 15 · 04 (DGM)
**Time:** ~60 minutes

## 问题

Research is an open-ended 任务. Unlike AlphaEvolve's algorithmic search or DGM's 基准-bounded self-modification, a 研究 结果 does 不have a machine-checkable 正确性 criterion. A 论文 is judged by 审查者, 不unit tests. That makes the 循环 harder to close — and more valuable if closed, because 研究 is 在哪里 compounding progress lives.

AI Scientist v1 (Sakana, 2024) closed the 循环 by starting 来自 人类-authored templates. The LLM filled in experiments within a fixed scaffolding. AI Scientist v2 (Yamada et al., 2025) removes the template requirement by using agentic tree search 带有 a vision-language 模型 critique 循环. The 系统 generates ideas, implements experiments, produces figures, writes a 论文, and iterates on 审查者 反馈.

Peer 审查 结论: one v2-generated 论文 was accepted at an ICLR 2025 workshop (带有 disclosure). 独立 评估 结论: the 系统 is far 来自 reliable. 两者 are true.

## 概念

### 这个architecture

1. **Idea generation.** The LLM proposes 研究 ideas conditioned on a topic and prior literature. v1 used templates; v2 uses agentic search over a space of hypotheses.
2. **Novelty check.** A literature retrieval step checks 是否 the idea has been published. This is the step 在哪里 Beel et al.'s 评估 found mislabeling — established methods frequently classified as novel.
3. **Experiment plan.** The 智能体 drafts an experimental protocol and writes code.
4. **Execution.** Code runs in a 沙箱. Failures are fed back 进入 a retry 循环. In Beel et al.'s measurements, 42% of experiments failed 来自 coding errors at this stage.
5. **Figure generation.** A vision-language 模型 reads generated figures and rewrites them for clarity. This was v2's key technical addition.
6. **Writeup.** The LLM drafts a 论文, iterates 带有 an 内部 审查者.
7. **Optional: submission.** The 论文 is submitted to a venue.

### 内容 the workshop-acceptance 结果 means

One v2-generated 论文 passed peer 审查 at an ICLR 2025 workshop. The authors disclosed the 论文's origin to the program committee. The acceptance is a data point; it is 不a license to 声明 the 系统 "does 研究."

Important 上下文: workshop papers are a lower bar than main-conference papers. Peer 审查 is noisy; a small fraction of submissions are accepted on 任何 给定 day. One success is a proof of concept, 不a 可靠性 声明. The Nature 2026 论文 documents the end-to-end 循环 and was itself co-authored by 人类 researchers; it is 不"the 系统 wrote a Nature 论文."

### 内容 the 独立 评估 found

Beel et al. (arXiv:2502.14297) ran an 外部 评估. Headline findings:

- **Experiment 失败.** 42% of experiments failed 来自 coding errors (bad imports, shape mismatches, undefined variables). The retry 循环 caught some, 不all.
- **Novelty mislabeling.** The literature-retrieval step frequently flagged established concepts as novel. This is the 研究 equivalent of hallucination.
- **Presentation-质量 缺口.** The vision-language figure critique produced publication-grade visuals, masking underlying experimental weaknesses.

这个last finding is the important one for this phase. A 系统 that produces convincing 输出 没有 doing convincing 研究 is more dangerous, 不safer, than one that fails obviously. Evaluation 必须 reach the underlying 声明, 不stop at the figure.

### 这个沙箱-escape concern

Sakana's own repository README warns:

> Due to the nature of this software, 哪个 executes LLM-generated code, we 不能 guarantee 安全. There are 风险 of dangerous packages, un控制led web access, and spawning of unintended 进程. Use at your own 风险 and consider Docker 隔离.

这is the operational shape of autonomy in an unverified domain. The LLM writes code; the code runs; the code can do anything the 进程 is allowed to do. 没有 a 沙箱 that hard-限制 文件系统, 网络, and 进程 actions, 任何 self-directed 研究 智能体 can exfiltrate data, burn 计算, or rewrite itself.

AlphaEvolve's 沙箱 story is easier because its 评估器 is tight. AI Scientist v2's 循环 runs open-ended code 带有 open-ended goals. That is 为什么 it needs stronger 隔离 (Docker minimum; seccomp / gVisor preferred) and a manual 审查 of 每个 submission 之前 it leaves the 系统.

### 位置 v2 sits in the frontier stack

| System | Target | 输出 kind | 评估器 | Known 失败 |
|---|---|---|---|---|
| AlphaEvolve | algorithms | code | unit + 基准 | bounded by 评估器 rigor |
| DGM | 智能体 scaffolding | code | SWE-bench | reward hacking |
| AI Scientist v2 | 研究 papers | text + code + figures | peer 审查 (weak) | experiment 失败, mislabeling, polish masking weakness |

v2 has the weakest automatic 评估器 of the three, the widest 输出 表面, and the shortest path to public artifacts. The operational 控制 (沙箱, 审查, disclosure) are doing most of the 安全 work.

## 使用它

`code/main.py` simulates the v2 循环 as a 说明 machine: idea → novelty check → experiment → figure → writeup → 审查 → accept-or-iterate. Each 说明 has a configurable 失败 probability pulled 来自 the Beel et al. findings. 运行 the simulator for N 循环 and count:

- 方式 many ideas reach submission.
- 方式 many submissions would have a critical experimental flaw the polished 论文 hides.
- 方式 retry 预算 trade off 质量 vs yield.

## 交付它

`outputs/skill-ai-scientist-sandbox-review.md` is a two-gate 审查 checklist for anything produced by a 研究-循环 智能体 之前 it leaves the 沙箱.

## 练习

1. 运行 `code/main.py` 带有 默认值 parameters. 内容 fraction of 循环 runs 生成 a "clean" 论文? 内容 fraction 生成 a 论文 带有 an experiment-失败 flaw the figure critique polished over?

2. The 默认值 already use Beel et al.'s 42% / 25%. Re-运行 带有 `--experiment-failure 0.20 --novelty-mislabel 0.10` and then 带有 `--experiment-failure 0.60 --novelty-mislabel 0.40`. 方式 does the polished-but-flawed share shift between the two runs?

3. 阅读 Sakana's AI Scientist v2 repo README on 沙箱 requirements. 命名 two additional restrictions (beyond Docker) you would apply for a multi-day 自主 运行.

4. 阅读 Beel et al. Section 4 on presentation-质量 缺口. 设计 one additional 评估器 that would catch polished-looking but experimentally flawed papers.

5. Propose a 人类-审查 protocol for 研究-智能体 输出 that scales better than "a PhD reads 每个 论文." 识别 the bottleneck and 设计 around it.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| AI Scientist v1 | "Sakana's templated 研究 智能体" | Filled experiments 进入 a fixed scaffold |
| AI Scientist v2 | "Template-free 研究 智能体" | Agentic tree search 带有 VLM figure critique |
| Agentic tree search | "Branching 研究 智能体" | Expands multiple experiment plans in parallel; prunes by 内部 critic |
| Vision-language critique | "VLM polish on figures" | 多模态 模型 reads figures and rewrites them for clarity |
| Literature retrieval | "Novelty check" | Searches prior work to 确认 idea novelty — documented to mislabel |
| Polish masking | "Pretty 论文, broken 研究" | Presentation 质量 exceeds experimental 质量; hides weaknesses |
| 沙箱 escape | "LLM code breaks out" | Agent-executed code does things the 循环 designer did 不intend |

## 延伸阅读

- [Yamada et al. (2025). The AI Scientist-v2](https://arxiv.org/abs/2504.08066) — 论文.
- [Sakana blog on the Nature 2026 publication](https://sakana.ai/ai-scientist-nature/) — vendor 摘要 带有 peer-审查 上下文.
- [Beel et al. (2025). Independent evaluation of The AI Scientist](https://arxiv.org/abs/2502.14297) — 外部 评估 numbers.
- [Sakana AI Scientist v1 paper](https://arxiv.org/abs/2408.06292) — the templated predecessor.
- [Anthropic — Measuring AI agent autonomy](https://www.anthropic.com/research/measuring-agent-autonomy) — broader framing of open-ended 研究 agents.
