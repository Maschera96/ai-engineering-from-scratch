# 奖励 Modeling & RLHF

> Humans cannot write a 奖励 函数 for "good 助手 响应," but they can compare two 响应 and pick the better one. Fit a 奖励模型 to those comparisons, then RL the 语言模型 against it. Christiano 2017. InstructGPT 2022. The recipe that turned GPT-3 into ChatGPT. In 2026 it is mostly being replaced by DPO — but the mental 模型 stays.

**类型：** Build
**语言：** Python
**先修：** Phase 5 · 05 (Sentiment), Phase 9 · 08 (PPO)
**时间：** 约 45 分钟

## 问题

你训练后的 a 语言模型 on the next-token-prediction 目标. It writes grammatical English. It also lies, rambles, and refuses to refuse. You cannot fix this with more pretraining — web 文本 is the problem, not the cure.

你want a *scalar 奖励* that says "响应 A is better than 响应 B for instruction X." Writing that 奖励 函数 by hand is impossible. "Helpfulness" is not a closed-form expression over 词元. But humans can compare two outputs and mark a preference. That is cheap to collect at 规模.

RLHF (Christiano et al. 2017; Ouyang et al. 2022) converts preferences into a 奖励模型, then optimizes the LM via PPO against that 奖励. In three 步骤: SFT → RM → PPO. It is the recipe that shipped ChatGPT, Claude, Gemini, and every other aligned-LLM in 2023–2025.

In 2026 the PPO 步骤 is mostly replaced by DPO (Phase 10 · 08) because it is cheaper and nearly as good for 对齐 tuning. But the *奖励模型* piece still underlies every Best-of-N 采样器, every RL-from-verifiable-rewards 流水线, and every 推理 模型 using a process 奖励模型. Understand RLHF and you understand the entire 对齐 stack.

## 概念

![Three-stage RLHF: SFT, RM training on pairwise prefs, PPO with KL penalty](../assets/rlhf.svg)

**Stage 1: Supervised 微调 (SFT).** Start from a 预训练 base 模型. Fine-tune on human-written demonstrations of the 目标 behavior (instruction-following 响应, helpful replies, etc.). Result: a 模型 `π_SFT` that is *biased toward good behavior* but still has an unbounded 动作 space.

**Stage 2: 奖励 模型 训练.**

- Collect pairs of 响应 `(y_+, y_-)` to prompts `x`, 标签ed by humans as "y_+ is preferred over y_-."
- 训练 a 奖励模型 `R_φ(x, y)` to assign higher scores to `y_+`.
- 损失: the **Bradley-Terry pairwise logistic**:

  `L(φ) = -E[ log σ(R_φ(x, y_+) - R_φ(x, y_-)) ]`

  σ is the sigmoid. The difference in 奖励 implies a log-odds of preference. BT has been the standard since 1952 (Bradley-Terry) and is the dominant choice in modern RLHF.

- `R_φ` is usually initialized from the SFT 模型 with a scalar 头 on top. Same transformer backbone; a single linear 层 outputs the 奖励.

**Stage 3: PPO against the RM with KL penalty.**

- Initialize the trainable 策略 `π_θ` from `π_SFT`. Keep a frozen *参考* `π_ref = π_SFT`.
- 奖励 at the end of a 响应 `y`:

  `r_total(x, y) = R_φ(x, y) - β · KL(π_θ(·|x) || π_ref(·|x))`

  The KL penalty prevents `π_θ` from drifting arbitrarily from `π_SFT` — it is a *regularizer*, not a hard trust region. `β` typically `0.01`-`0.05`.
- 运行PPO (Lesson 08) with this 奖励. Advantages are computed on the token-level 轨迹, but the RM scores only the full 响应.

**Why the KL?** Without it, PPO will happily find reward-hacking strategies — the RM was only 训练后的 on in-distribution completions. An out-of-distribution 响应 might 分数 higher than any human-written one. The KL keeps `π_θ` near the manifold where the RM was 训练后的. It is the single most important knob in RLHF.

**2026 status:**

- **DPO** (Rafailov 2023): closed-form algebra collapses Stage 2+3 into a single supervised 损失 over preference 数据. No RM, no PPO. Same 质量 on 对齐 benchmarks for a fraction of the 计算. Covered in Phase 10 · 08.
- **GRPO** (DeepSeek 2024–2025): PPO with a group-relative 基线 instead of a critic, 奖励 from a *verifier* (code runs / math 答案 matches) instead of a human-trained RM. Dominant for 推理 模型. Covered in Phase 9 · 12.
- **Process 奖励模型s (PRMs):** 分数 partial solutions (each 推理 步骤), used in both RLHF and GRPO variants for 推理.
- **宪法 AI / RLAIF:** use an aligned LLM to 生成 preferences instead of humans. Scales the preference 预算.

## 动手构建

这lesson uses tiny synthetic "prompts" and "响应" represented as strings. The RM is a linear scorer over a bag-of-tokens representation. No 真实 LLM — the *shape* of the 流水线 matters, not the 规模. See `code/main.py`.

### 步骤 1: synthetic preference 数据

```python
PROMPTS = ["help me", "answer me", "explain this"]
GOOD_WORDS = {"clear", "specific", "kind", "thorough"}
BAD_WORDS = {"vague", "rude", "wrong", "short"}

def make_pair(rng):
    x = rng.choice(PROMPTS)
    y_good = rng.choice(list(GOOD_WORDS)) + " " + rng.choice(list(GOOD_WORDS))
    y_bad = rng.choice(list(BAD_WORDS)) + " " + rng.choice(list(BAD_WORDS))
    return (x, y_good, y_bad)
```

In 真实 RLHF this is replaced by human 标签ers. The shape — `(prompt, preferred_response, rejected_response)` — is identical.

### 步骤 2: Bradley-Terry 奖励模型

Linear 分数: `R(x, y) = w · bag(y)`. 训练 to minimize the BT pairwise log-loss:

```python
def rm_train_step(w, x, y_pos, y_neg, lr):
    r_pos = dot(w, bag(y_pos))
    r_neg = dot(w, bag(y_neg))
    p = sigmoid(r_pos - r_neg)
    for tok, cnt in bag(y_pos).items():
        w[tok] += lr * (1 - p) * cnt
    for tok, cnt in bag(y_neg).items():
        w[tok] -= lr * (1 - p) * cnt
```

After a few hundred updates, `w` assigns positive 权重 to good-word 词元 and negative to bad.

### 步骤 3: PPO-like 策略 on top of RM

Our toy 策略 produces a single 词元 from a 词表. We 分数 the 词元 under the RM, 计算 `log π_θ(token | prompt)`, add a KL-to-reference penalty, and apply the clipped PPO surrogate.

```python
def rlhf_step(theta, ref, w, prompt, rng, eps=0.2, beta=0.1, lr=0.05):
    logits_theta = policy_logits(theta, prompt)
    probs = softmax(logits_theta)
    token = sample(probs, rng)
    logits_ref = policy_logits(ref, prompt)
    probs_ref = softmax(logits_ref)
    reward = dot(w, bag([token])) - beta * kl(probs, probs_ref)
    # ppo-style update on theta, treating reward as the return
    ...
```

### 步骤 4: monitor the KL

Track mean `KL(π_θ || π_ref)` every update. If it creeps past `~5-10` the 策略 has drifted far from `π_SFT` — lower `β` is rising or 奖励 hacking is starting. This is the top diagnostic in 真实 RLHF.

### 步骤 5: the 生产 recipe with TRL

Once you understand the toy 流水线, here is the same 循环 as a 真实 library 用户 writes it. Hugging Face's [TRL](https://huggingface.co/docs/trl) is the 参考 implementation — `RewardTrainer` for Stage 2 and `PPOTrainer` (with a KL-to-reference built in) for Stage 3.

```python
# Stage 2: reward model from pairwise preferences
from trl import RewardTrainer, RewardConfig
from transformers import AutoModelForSequenceClassification, AutoTokenizer

tok = AutoTokenizer.from_pretrained("meta-llama/Llama-3.1-8B-Instruct")
rm = AutoModelForSequenceClassification.from_pretrained(
    "meta-llama/Llama-3.1-8B-Instruct", num_labels=1
)

# dataset rows: {"prompt", "chosen", "rejected"} — Bradley-Terry format
trainer = RewardTrainer(
    model=rm,
    tokenizer=tok,
    train_dataset=preference_data,
    args=RewardConfig(output_dir="./rm", num_train_epochs=1, learning_rate=1e-5),
)
trainer.train()
```

```python
# Stage 3: PPO against the RM with KL penalty to the SFT reference
from trl import PPOTrainer, PPOConfig, AutoModelForCausalLMWithValueHead

policy = AutoModelForCausalLMWithValueHead.from_pretrained("./sft-checkpoint")
ref    = AutoModelForCausalLMWithValueHead.from_pretrained("./sft-checkpoint")  # frozen

ppo = PPOTrainer(
    config=PPOConfig(learning_rate=1.41e-5, batch_size=64, init_kl_coef=0.05,
                     target_kl=6.0, adap_kl_ctrl=True),
    model=policy, ref_model=ref, tokenizer=tok,
)

for batch in dataloader:
    responses = ppo.generate(batch["query_ids"], max_new_tokens=128)
    rewards   = rm(torch.cat([batch["query_ids"], responses], dim=-1)).logits[:, 0]
    stats     = ppo.step(batch["query_ids"], responses, rewards)
    # stats includes: mean_kl, clip_frac, value_loss — the three PPO diagnostics
```

Three things the library does for you. `adap_kl_ctrl=True` implements the adaptive-β 调度: if observed KL exceeds `target_kl`, β doubles; if below half, β halves. The 参考 模型 is frozen by convention — you must not accidentally share 参数 with `policy`. And the value 头 lives on the same backbone as the 策略 (`AutoModelForCausalLMWithValueHead` attaches a scalar MLP 头), which is why TRL reports `policy/kl` and `value/loss` separately.

## Pitfalls

- **Over-optimization / 奖励 hacking.** The RM is imperfect; `π_θ` finds adversarial completions that 分数 high but are bad. Symptoms: 奖励 climbs indefinitely while human 评估 分数 plateaus or drops. 修复: stop early, raise `β`, broaden RM 训练 数据.
- **Length hacking.** RMs 训练后的 on helpful 响应 often implicitly 奖励 length. The 策略 learns to pad 响应. Remediation: length-normalized 奖励, or RLAIF with a length-aware RM.
- **Too-small RM.** The RM needs to be at least as large as the 策略. A tiny RM cannot faithfully 分数 the 策略's outputs.
- **KL tuning.** Too low β → drift and 奖励 hacking. Too high β → 策略 barely changes. The standard trick is an *adaptive* β that 目标 a fixed KL per 步骤.
- **Preference-data 噪声.** ~30% of human 标签s are noisy or ambiguous. Calibrate by 训练 the RM on agreement-filtered 数据 or use a temperature on BT.
- **Off-policy problems.** PPO 数据 is slightly off-policy after the first 轮次. Monitor clip fraction as in Lesson 08.

## 实际使用

RLHF in 2026 is layered:

|层|目标|Method|
|-------|--------|--------|
|Instruction following, helpfulness, harmlessness|对齐|DPO (Phase 10 · 08) preferred over RLHF-PPO.|
|推理 correctness (math, code)|Capability|GRPO with verifier 奖励 (Phase 9 · 12).|
|Long-horizon multi-step tasks|Agentic|PPO / GRPO with process 奖励模型s over 步骤.|
|安全 / refusal behavior|安全|RLHF-PPO with separate 安全 RM, or 宪法 AI.|
|Best-of-N at 推理|Fast 对齐|Use RM at decode time; no 策略 训练 needed.|
|奖励 distillation|推理 计算|训练 a small "奖励 头" on top of a frozen LM.|

RLHF was *the* method in 2022–2024. In 2026, 生产 对齐 pipelines are DPO-first, PPO-only for the RM-intensive or safety-critical 步骤.

## 交付成果

Save as `outputs/skill-rlhf-architect.md`:

```markdown
---
name: rlhf-architect
description: Design an RLHF / DPO / GRPO alignment pipeline for a language model, including RM, KL, and data strategy.
version: 1.0.0
phase: 9
lesson: 9
tags: [rl, rlhf, alignment, llm]
---

Given a base LM, a target behavior (alignment / reasoning / refusal / agent), and a preference or verifier budget, output:

1. Stage. SFT? RM? DPO? GRPO? With justification.
2. Preference or verifier source. Humans, AI feedback, rule-based, unit-test-pass, or reward distillation.
3. KL strategy. Fixed β, adaptive β, or DPO (implicit KL).
4. Diagnostics. Mean KL, reward stability, over-optimization guard (holdout human eval).
5. Safety gate. Red-team set, refusal rate, safety RM separate from helpfulness RM.

Refuse to ship RLHF-PPO without a KL monitor. Refuse to use an RM smaller than the target policy. Refuse length-only rewards. Flag any pipeline that does not hold back a blind human-eval set as lacking over-optimization protection.
```

## 练习

1. **Easy.** 训练 the Bradley-Terry 奖励模型 in `code/main.py` on 500 synthetic preference pairs. Measure pairwise accuracy on a held-out 100 pairs. Should exceed 90%.
2. **Medium.** Run the toy PPO-RLHF 循环 with `β ∈ {0.0, 0.1, 1.0}`. For each, plot RM 分数 vs KL-to-reference over updates. Which runs reward-hack?
3. **Hard.** Implement DPO (closed-form preference-likelihood 损失) on the same preference 数据 and compare to the RLHF-PPO 流水线 in 计算 used and final RM 分数 achieved.

## Key Terms

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|RLHF|"对齐 RL"|Three-stage SFT + RM + PPO 流水线 (Christiano 2017, Ouyang 2022).|
|奖励 模型 (RM)|"The scoring net"|Learned scalar 函数 fit to pairwise preferences via Bradley-Terry.|
|Bradley-Terry|"Pairwise logistic 损失"|`P(y_+ ≻ y_-) = σ(R(y_+) - R(y_-))`; the standard RM 目标.|
|KL penalty|"Stay near the 参考"|`β · KL(π_θ \|\|π_ref)` in the 奖励; the anti-reward-hacking regularizer.|
|奖励 hacking|"Goodhart's law"|策略 exploits RM flaws; symptoms: 奖励 up, human 评估 flat.|
|RLAIF|"AI-标签ed preferences"|RLHF where 标签s come from another LM instead of humans.|
|PRM|"Process 奖励 模型"|Scores partial 推理 步骤; used in 推理 pipelines.|
|宪法 AI|"Anthropic's method"|AI-generated preferences guided by explicit rules.|

## 延伸阅读

- [Christiano et al. (2017). Deep Reinforcement Learning from Human Preferences](https://arxiv.org/abs/1706.03741) — the paper that started RLHF.
- [Ouyang et al. (2022). InstructGPT — Training language models to follow instructions with human feedback](https://arxiv.org/abs/2203.02155) — the recipe behind ChatGPT.
- [Stiennon et al. (2020). Learning to summarize with human feedback](https://arxiv.org/abs/2009.01325) — earlier RLHF for summarization.
- [Rafailov et al. (2023). Direct Preference Optimization](https://arxiv.org/abs/2305.18290) — DPO; the post-RLHF default in 2026.
- [Bai et al. (2022). Constitutional AI: Harmlessness from AI Feedback](https://arxiv.org/abs/2212.08073) — RLAIF and self-critique 循环.
- [Anthropic RLHF paper (Bai et al. 2022). Training a Helpful and Harmless Assistant](https://arxiv.org/abs/2204.05862) — the HH paper.
- [Hugging Face TRL library](https://huggingface.co/docs/trl) — 生产 `RewardTrainer` and `PPOTrainer`. Read the trainer 来源 for the adaptive-KL and value-head details.
- [Hugging Face — Illustrating Reinforcement Learning from Human Feedback](https://huggingface.co/blog/rlhf) by Lambert, Castricato, von Werra, Havrilla — the canonical walk-through of the three-stage 流水线 with diagrams.
- [von Werra et al. (2020). TRL: Transformer Reinforcement Learning](https://github.com/huggingface/trl) — the library; `examples/` has end-to-end RLHF scripts for Llama, Mistral, and Qwen.
- [Sutton & Barto (2018). Ch. 17.4 — Designing Reward Signals](http://incompleteideas.net/book/RLbook2020.pdf) — the reward-hypothesis view; essential prerequisite for thinking about 奖励 hacking.
