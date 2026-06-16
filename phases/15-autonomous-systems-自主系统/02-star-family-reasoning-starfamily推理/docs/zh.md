# STaR、V-STaR、Quiet-STaR：自学推理

> The smallest possible self-improvement 循环 sits 内部 the 理由. A 模型 generates a chain of thought, keeps the ones that land on correct answers, and fine-tunes on those. That is STaR. V-STaR adds a verifier so 推理-time selection is better. Quiet-STaR pushes the 理由 down to 每个 词元. 所有 three work. None of them are magic — the 循环 preserves 任何 shortcut that happened to reach the right answer.

**Type:** Learn
**Languages:** Python (stdlib, bootstrap-loop simulator)
**Prerequisites:** Phase 13 · 01-03 (Reasoning and CoT), Phase 15 · 01 (long-horizon framing)
**Time:** ~60 minutes

## 问题

这个straightforward way to teach a 模型 to reason is to collect 人类-written reasoning traces. That is expensive, slow, and bounded by 如何 much high-质量 chain-of-thought 人类 are willing to 编写.

STaR (Self-Taught Reasoner, Zelikman et al., 2022) asks: 什么 if the 模型 writes its own rationales and grades them 针对 known answers? The 循环 is:

1. Sample a reasoning trace plus answer.
2. If the final answer is correct, keep the trace.
3. Fine-tune on the kept traces.
4. Repeat.

It works. GSM8K and CommonsenseQA 两者 improved 没有 new 人类 annotation. But the 循环 has a built-in bias: 任何 理由 that produced the right answer is retained, regardless of 是否 the reasoning itself was sound. V-STaR (Hosseini et al., 2024) patches this 带有 a learned verifier; Quiet-STaR (Zelikman et al., 2024) generalizes the idea to per-词元 内部 rationales.

## 概念

### STaR: bootstrap on 什么 worked

Start 来自 a base 模型 带有 some weak reasoning ability. On each training problem, sample a 理由 plus answer. If the answer matches the label, keep the (problem, 理由, answer) triple. Fine-tune the 模型 on the kept 设置. Repeat.

One twist matters. If the 模型 can never get a problem right, the 循环 不能 learn on it. STaR adds **rationalization**: for problems the 模型 fails, inject the correct answer as a hint and re-提示词 the 模型 to 生成 a 理由 that leads to it. Rationalized rationales are added to the training 设置.

Result in the original 论文 (Zelikman et al., 2022): a GPT-J base 模型 improved on GSM8K 来自 5.8% to 10.7% through repeated STaR rounds 带有 rationalization — about 5 percentage points absolute. On CommonsenseQA, STaR-trained GPT-J 6B reached 72.5%, comparable to a fine-tuned GPT-3 175B (~73%) — a roughly 30x larger 模型 trained on hand-annotated rationales.

### V-STaR: train a verifier 带有 DPO

STaR throws away incorrect rationales. Hosseini et al. (2024) observed those are also data: 每个 pair of (理由, "is this correct") can train a verifier. They use Direct Preference Optimization over 两者 correct and incorrect solutions to build a ranker. At 推理 time, sample N rationales and pick the verifier's top choice.

Reported delta: +4 to +17 percentage points over prior self-improvement baselines on GSM8K and MATH, 带有 most of the gain coming 来自 using the verifier for 推理-time selection rather than for additional generator fine-tuning.

### Quiet-STaR: per-词元 内部 rationales

Zelikman et al. (2024) asked: 什么 if the 模型 learns to generate a short 内部 理由 at 每个 词元 position, 不just between problem and answer? Quiet-STaR trains a 模型 to emit a hidden "thought" 之前 each predicted 词元, then mixes the thought-aware prediction 带有 the baseline prediction via a learned weight.

Result: Mistral 7B gained absolute zero-shot improvements on GSM8K from 5.9% to 10.9% and CommonsenseQA from 36.3% to 47.2% 没有 任务-具体 fine-tuning. The model learned "when to think" — hard tokens get longer 内部 rationales; easy ones get almost none.

### 原因 所有 three share a 安全 concern

所有 three methods use the final answer as the gradient signal. A 理由 that reaches the right answer via flawed reasoning — exploiting a shortcut, guessing, or using a non-generalizing pattern — gets positively reinforced. On in-分布 problems the shortcut works. On out-of-分布 problems it breaks 静默地.

V-STaR's verifier mitigates by learning to rank rationales, but the verifier is trained on the same label 设置. It can learn to prefer well-formatted wrong reasoning over honest uncertainty. The safer 设计 is to combine STaR-style data 带有 (a) 进程-supervised reward models (rewarding intermediate steps, 不just answers) and (b) 留出 OOD 评估 that breaks simple shortcuts.

### 对比

| 方法 | Training signal | 推理 成本 | Data waste | Known 失败 mode |
|---|---|---|---|---|
| STaR | keep (理由, answer) if correct | 1x | discards 所有 incorrect rationales | shortcut rationales |
| STaR + rationalization | above + correct-answer hinted retries | 1x | less | rationalized rationales may be implausible |
| V-STaR | STaR + DPO verifier 来自 两者 类别 | Nx (best-of-N) | minimal | verifier can reinforce confident wrongness |
| Quiet-STaR | per-词元 理由 + mixing weight | 1.5-3x | minimal | still answer-conditioned gradient |

### 位置 this sits in the 2026 stack

STaR is old. But the pattern reappears everywhere in 2025-2026. RL on verifiable math problems (DeepSeek-R1, Kimi-k1.5, o1) is STaR's answer-conditioned gradient signal, scaled up. 进程 reward models (Lightman et al., 2023; OpenAI's "Let's 验证 step by step") are the 进程-supervised alternative. AlphaEvolve (Lesson 3) is STaR for code, 带有 a program 评估器 instead of a label. Darwin Godel Machine (Lesson 4) is STaR for the 智能体 scaffolding itself.

Understanding STaR makes 所有 of these click. It is the minimum-viable self-improvement 循环.

```figure
reflection-loop
```

## 使用它

`code/main.py` runs a simulated STaR 循环 on a toy arithmetic 任务. You can watch:

- 方式 accuracy climbs over bootstrap rounds.
- 方式 shortcuts sneak in: the simulator includes a "lazy" 理由 类别 that gets the right answer 40% of the time but generalizes badly. Watch 是否 STaR keeps them.
- 方式 a verifier (V-STaR style) helps at 推理 but 不能 fully prune shortcuts introduced 期间 training.

## 交付它

`outputs/skill-star-loop-reviewer.md` helps you 审计 a 拟议的 self-taught-reasoning 流水线 之前 you train on it.

## 练习

1. 运行 the simulator. 设置 the shortcut frequency to zero, then to 0.4. 方式 much does final accuracy diverge between the two runs, even though 两者 hit >90% on the training 分布?

2. Add a 留出 OOD test to the simulator. Draw problems 来自 a different 分布 and evaluate the bootstrapped 模型 on 两者 in-分布 and OOD sets. Quantify the 缺口.

3. 阅读 the Quiet-STaR 论文 (arXiv:2403.09629) Section 3. 解释 the "end-of-thought" 词元 and the mixing-weight head in three sentences each.

4. 比较 STaR's keep-if-correct filter to a 进程-supervised alternative that rewards each 理由 step independently. 识别 the labelling 成本 difference and the plausible 质量 difference.

5. 设计 one 评估 that would catch shortcut rationales in a deployed 模型. It does 不have to be perfect — it has to break the simplest shortcuts a STaR 循环 would reinforce.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| STaR | "Self-Taught Reasoner" | Fine-tune on 模型-generated rationales that land correct answers; repeat |
| Rationalization | "Hinted retry" | Inject the correct answer and re-提示词 for a 理由 on problems the base 模型 fails |
| V-STaR | "Verifier STaR" | DPO-train a verifier on 两者 correct and incorrect rationales, use it for 推理-time selection |
| Quiet-STaR | "Per-词元 rationales" | Generate hidden thoughts at 每个 词元 position; mix 带有 baseline prediction |
| Answer-conditioned gradient | "Outcome-based signal" | 这个training 循环 rewards final answers, 不reasoning steps |
| 进程 reward 模型 | "Step-level verifier" | Reward 模型 trained on per-step 正确性, 不outcome — contrasts 带有 STaR |
| Shortcut 理由 | "Right answer, wrong reasoning" | 一个rationale that reaches the label via a non-generalizing pattern; STaR keeps these |

## 延伸阅读

- [Zelikman et al. (2022). STaR: Bootstrapping Reasoning With Reasoning](https://arxiv.org/abs/2203.14465) — the original 论文.
- [Hosseini et al. (2024). V-STaR: Training Verifiers for Self-Taught Reasoners](https://arxiv.org/abs/2402.06457) — adds a DPO verifier for 推理-time selection.
- [Zelikman et al. (2024). Quiet-STaR: Language Models Can Teach Themselves to Think Before Speaking](https://arxiv.org/abs/2403.09629) — per-词元 内部 rationales.
- [Lightman et al. (2023). Let's Verify Step by Step](https://arxiv.org/abs/2305.20050) — 进程 reward models, the alternative gradient signal.
- [DeepSeek-R1 paper (arXiv:2501.12948)](https://arxiv.org/abs/2501.12948) — RL on verifiable 任务, STaR scaled to frontier training.
