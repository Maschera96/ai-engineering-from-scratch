# 作为对齐信号的指令遵循

> Every later critique of RLHF argues against this pipeline. Before you study how optimization pressure distorts a proxy, you have to see the proxy. InstructGPT (Ouyang et al., 2022) defined the reference architecture: 监督微调 on instruction-response pairs, a 奖励模型 trained on pairwise 偏好 rankings, and PPO against the 奖励模型 with a KL penalty to the SFT 策略. A 1.3B InstructGPT was preferred over a 175B GPT-3. That single result is the reason every frontier lab in 2026 still ships an RLHF-shaped post-training pipeline.

**类型:** 学习
**语言:** Python (stdlib, toy three-stage pipeline)
**先修要求:** Phase 10 · 06 (SFT), Phase 10 · 07 (RLHF), Phase 10 · 08 (DPO)
**时间:** ~45 分钟

## 学习目标

- 说出 the three stages of the InstructGPT pipeline and the loss used in each.
- 解释 why a 1.3B instruction-tuned 模型 beat the raw 175B GPT-3 on 人类偏好 评估.
- 说明 what the KL penalty in stage 3 is protecting against and why removing it collapses to mode-seeking behaviour.
- 描述 the 对齐 tax and the PPO-ptx mitigation Ouyang et al. used against it.

## 问题

预训练 language models complete text. They do not answer questions. Ask GPT-3 "write a Python function that reverses a list" and you often get back another prompt, because most of the training distribution is web text that continues with more web text. The 模型 is doing its job ， the job is wrong.

The proxy every serious lab used to fix this is 人类偏好. Two completions go to a rater; the rater picks the better one; a 奖励模型 learns the rater. Then an RL loop shifts the 策略 toward outputs the 奖励模型 scores high. That is the full InstructGPT thesis in three sentences. The rest of the paper is engineering.

## 概念

### 阶段 1：监督微调 (SFT)

Collect prompt-response pairs where the response is what a well-intentioned human would write. Ouyang et al. used 13k prompts from labelers and the OpenAI API. 微调 the base 模型 on this data with standard cross-entropy loss.

What SFT gives you: the 模型 now answers questions instead of continuing them. What it does not give you: any signal about which answer the rater prefers when multiple are plausible.

### 阶段 2：奖励模型 (RM)

For each prompt, sample K completions from the SFT 模型. A 标注员 ranks them. Train a 奖励模型 that scores any prompt-response pair so that, for pairs where `y_w` was preferred over `y_l`:

```
L_RM = -log sigmoid(r(x, y_w) - r(x, y_l))
```

This is the Bradley-Terry pairwise 偏好 loss. The RM is usually initialized from the SFT 模型 with the LM head replaced by a scalar head.

Reward models are small: 6B was enough for the 175B InstructGPT. They are also fragile ， section 5 of the paper is mostly about reward-hacking behaviours that showed up at small scale.

### 阶段 3：带 KL 惩罚的 PPO

Define the objective:

```
J(pi) = E_{x~D, y~pi(.|x)} [ r(x, y) ] - beta * KL(pi(.|x) || pi_SFT(.|x))
```

Maximize with PPO. The KL term keeps `pi` from drifting far from the SFT 策略. Without it, the optimizer finds adversarial examples ， strings that score high under the RM because the RM never saw them, not because humans actually prefer them.

The KL coefficient `beta` is the single most important RLHF hyperparameter. Too low: 奖励黑客. Too high: no improvement over SFT.

### 对齐税

After RLHF, the 模型 is preferred by humans but regresses on standard benchmarks (SQuAD, HellaSwag, DROP). Ouyang et al. call this the 对齐 tax and fix it with PPO-ptx: mix 预训练 gradients into the RL objective so the 模型 does not forget how to do downstream tasks it was never rewarded for.

```
J_ptx(pi) = J(pi) + gamma * E_{x~D_pretrain} [ log pi(x) ]
```

PPO-ptx became standard. Anthropic, DeepMind, and Meta all use some variant.

### 结果

A 1.3B InstructGPT (SFT + RM + PPO-ptx) is preferred by labelers over the 175B base GPT-3 about 70% of the time. The gap widens on hidden-test prompts from production traffic. Two things to read off this number:

1. 对齐 is a different axis from 能力. The 175B 模型 had more 能力; the 1.3B 模型 had more 对齐; labelers preferred the aligned one.
2. The 能力 floor is set by the base 模型. You cannot RLHF a base 模型 into knowing facts it never saw.

### 为什么这是 Phase 18 的参照点

Every critique in later lessons ， 奖励黑客 (Lesson 2), DPO (Lesson 3), 迎合 (Lesson 4), CAI (Lesson 5), 休眠智能体 (Lesson 7), 对齐 faking (Lesson 9) ， argues against some part of this pipeline. 奖励黑客 attacks stage 2. DPO collapses stages 2 and 3. CAI replaces the human 标注员. 迎合 shows the 标注员 is a biased signal. 对齐 faking shows the 策略 can route around stage 3 entirely. You cannot follow any of these critiques without the pipeline in your head first.

## 使用它

`code/main.py` simulates the three stages on toy 偏好 data. The base "策略" is a biased coin over actions {A, B, C}. Stage 1 SFT mimics 标注员 actions on 200 prompts. Stage 2 fits a Bradley-Terry 奖励模型 from 500 pairwise rankings. Stage 3 runs a simplified PPO update with a KL penalty to the SFT 策略. You can watch the reward climb, the KL divergence grow, and the 策略 drift ， and you can turn off the KL term to see 奖励黑客 appear inside 50 update steps.

观察重点:

- Reward trajectory with `beta = 0.1` vs `beta = 0.0`.
- KL(pi || pi_SFT) over training steps.
- Final action distribution compared to 标注员 偏好.

## 交付它

本课产出 `outputs/skill-instructgpt-explainer.md`. 给定 an RLHF pipeline description or a paper abstract, it identifies which of the three stages is being modified, what loss is being used at each stage, and whether a KL penalty or equivalent regularizer is present.

## 练习

1. 运行 `code/main.py`. Set `beta = 0.0` and report the action distribution after 200 PPO steps. 解释 the mode-seeking behaviour in one paragraph.

2. 修改 the 奖励模型 to have a +0.5 偏差 for action B (a simulated reward bug). 运行 PPO with `beta = 0.1`. Does the KL penalty prevent the 策略 from exploiting the 偏差? At what `beta` does exploitation become visible?

3. 阅读 Ouyang et al. (arXiv:2203.02155) Figure 1. Reproduce the 标注员-偏好 curve by running PPO for 1, 5, 20, 100 steps and measuring 偏好 against the SFT 模型.

4. The paper's Section 4.3 reports a 1.3B InstructGPT beats 175B GPT-3 about 70% of the time. Why would the ratio be higher on hidden production prompts than on the 标注员's own prompts?

5. 替换 the PPO loss with DPO (Phase 10 · 08) on the same 偏好 data. 比较 final 策略 drift (KL to SFT) and final reward. Which method drifts further at matched reward?

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|-----------------|------------------------|
| SFT | "instruction tuning" | Stage 1: cross-entropy 微调 on prompt-response pairs |
| 奖励模型 | "the RM" | Scalar regressor over (prompt, response) trained with Bradley-Terry on pairwise labels |
| Bradley-Terry | "pairwise 偏好 loss" | -log sigmoid(r_w - r_l); reduces pairwise ranking to binary classification |
| KL penalty | "the regularizer" | `beta * KL(pi \|\| pi_SFT)` ， keeps the RL 策略 near the SFT anchor |
| PPO-ptx | "PPO with pretraining mix" | Adds a fraction of 预训练 log-likelihood to the PPO objective to offset the 对齐 tax |
| 对齐 tax | "the RLHF regression" | Post-RLHF drop on standard benchmarks that RLHF did not target |
| 标注员 偏好 | "the ground truth" | Sample of human rankings; the RM is a statistical proxy for this, not for "human values" |

## 延伸阅读

- [Ouyang et al. ， Training language models to follow instructions with human feedback (arXiv:2203.02155)](https://arxiv.org/abs/2203.02155) ， the InstructGPT paper, foundation for every RLHF pipeline that followed
- [Stiennon et al. ， Learning to summarize from human feedback (arXiv:2009.01325)](https://arxiv.org/abs/2009.01325) ， the RLHF-for-summarization predecessor
- [Christiano et al. ， Deep reinforcement learning from human preferences (arXiv:1706.03741)](https://arxiv.org/abs/1706.03741) ， the original 偏好-based RL formulation
- [Bai et al. ， Training a Helpful and Harmless Assistant with RLHF (arXiv:2204.05862)](https://arxiv.org/abs/2204.05862) ， Anthropic's HH extension of the InstructGPT pipeline
