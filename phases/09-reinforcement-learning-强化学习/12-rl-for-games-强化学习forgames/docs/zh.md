# RL for Games — AlphaZero, MuZero, and the LLM-Reasoning Era

> 1992: TD-Gammon beat human champions at backgammon with pure TD. 2016: AlphaGo beat Lee Sedol. 2017: AlphaZero dominated chess, shogi, and Go from scratch. 2024: DeepSeek-R1 proved the same recipe, with GRPO replacing PPO, works on 推理. Games are the 基准 that drives every breakthrough in this phase.

**类型：** Build
**语言：** Python
**先修：** Phase 9 · 05 (DQN), Phase 9 · 08 (PPO), Phase 9 · 09 (RLHF), Phase 9 · 10 (MARL)
**时间：** 约 120 分钟

## 问题

Games have everything RL wants. Clean 奖励 (win/损失). Infinite episodes (self-play resets). Perfect simulation (the game *is* the simulator). Discrete or small continuous 动作 spaces. Multi-agent structure that forces adversarial robustness.

And games are how every major RL breakthrough was tested. TD-Gammon (backgammon, 1992). Atari-DQN (2013). AlphaGo (2016). AlphaZero (2017). OpenAI Five (Dota 2, 2019). AlphaStar (StarCraft II, 2019). MuZero (learned 模型, 2019). AlphaTensor (matrix multiplication, 2022). AlphaDev (sorting algorithms, 2023). DeepSeek-R1 (math 推理, 2025) — the latest demonstration that game-RL techniques work on 文本.

这capstone surveys the three landmark architectures — AlphaZero, MuZero, and GRPO — through a single unifying lens: **self-play + search + 策略 improvement**. Each generalizes the previous; GRPO in particular is AlphaZero's recipe applied to LLM 推理, with 词元 as actions and mathematical verification as the win 信号.

## 概念

![AlphaZero ↔ MuZero ↔ GRPO: same loop, different environments](../assets/rl-games.svg)

**The unifying 循环.**

```text
while True:
    trajectory = self_play(current_policy, search)     # play game against self
    policy_target = search.improved_policy(trajectory) # search improves raw policy
    policy_net.update(policy_target, value_target)     # supervised on search output
```

**AlphaZero (2017).** Silver et al. Given a game (chess, shogi, Go) with known rules:

- Policy-value network: one tower `f_θ(s) → (p, v)`. `p` is a 先验 over legal moves. `v` is the expected game outcome.
- Monte Carlo Tree Search (MCTS): at each move, expand a tree of possible continuations. Use `(p, v)` as the 先验 + bootstrap. Select nodes by UCB (PUCT): `a* = argmax Q(s, a) + c · p(a|s) · √N(s) / (1 + N(s, a))`.
- Self-play: play games agent-vs-agent. At move `t`, the MCTS visit 分布 `π_t` becomes the 策略 训练 目标.
- 损失: `L = (v - z)² - π · log p + c · ||θ||²`. `z` is the game outcome (+1 / 0 / -1).

Zero human knowledge. Zero handcrafted heuristics. A single recipe that mastered chess, shogi, and Go after a few tens of millions of self-play games each.

**MuZero (2019).** Schrittwieser et al. Removes the requirement that the rules are known.

- Instead of a fixed 环境, learn a *潜变量 dynamics 模型* `(h, g, f)`:
  - `h(s)`: encode observation to 潜变量 状态.
  - `g(s_latent, a)`: 预测 next 潜变量 状态 + 奖励.
  - `f(s_latent)`: 预测 策略 先验 + value.
- MCTS runs in the *learned 潜变量 space*. Same search, same 训练 循环.
- Works on Go, chess, shogi *and* Atari — one algorithm, no rule knowledge.

**Stochastic MuZero (2022).** Adds stochastic dynamics and chance nodes; extends to backgammon-class games.

**Muesli, Gumbel MuZero (2022-2024).** Improvements on 样本 efficiency and deterministic search.

**GRPO (2024-2025).** DeepSeek-R1 recipe. Same AlphaZero-shaped 循环, applied to language-model 推理:

- "Game": 答案 a math / coding / 推理 problem. "Win" = verifier (test case passes, numerical 答案 matches) returns 1.
- 策略: the LLM. Actions: 词元. 状态: 提示词 + response-so-far.
- No critic (PPO-style V_φ). Instead, for each 提示词, 样本 `G` completions from the 策略. 计算 奖励 for each. Use the **group-relative advantage** `A_i = (r_i - mean_r) / std_r` as the 信号 for REINFORCE-style update.
- KL penalty to 参考 策略 to prevent drift (like RLHF).
- Full 损失:

  `L_GRPO(θ) = -E_{q, {o_i}} [ (1/G) Σ_i A_i · log π_θ(o_i | q) ] + β · KL(π_θ || π_ref)`

No 奖励模型, no critic, no MCTS. Group-relative 基线 replaces all three. Matches or exceeds PPO-RLHF 质量 on 推理 benchmarks at a fraction of the 计算.

**The R1 recipe in full.** DeepSeek-R1 (DeepSeek 2025) is two 模型 in one paper:

- **R1-Zero.** Start from the DeepSeek-V3 base 模型. No SFT. Apply GRPO directly with two 奖励 components: *accuracy 奖励* (rule-based — did the final 答案 parse to the correct number / did the code pass unit tests) and *format 奖励* (did the completion wrap its chain-of-thought in `<think>…</think>` tags). Over thousands of 步骤, average 响应 length grows from ~100 to ~10,000 词元 and math 基准 scores climb to near-o1-preview levels. The 模型 learns to 原因 from scratch. The downside: its chains of 思考 are often unreadable, mix languages, and lack stylistic polish.
- **R1.** 修复 R1-Zero's readability problems with a four-stage 流水线:
  1. **Cold-start SFT.** Collect a few thousand long-CoT demonstrations with clean formatting. Supervised-finetune the base 模型 on them. This gives a readable starting point.
  2. **Reasoning-oriented GRPO.** Apply GRPO with the accuracy+format rewards plus a *language-consistency* 奖励 to prevent code-switching.
  3. **Rejection 采样 + SFT round 2.** 样本 ~600K 推理 trajectories from the RL checkpoint, keep only those with correct final answers and readable CoT, and combine with ~200K non-reasoning SFT examples (writing, QA, self-cognition). Fine-tune the base again.
  4. **Full-spectrum GRPO.** One more RL round covering both 推理 (rule-based rewards) and general 对齐 (helpfulness/harmlessness preference-based rewards).

这个result matches o1 on AIME and MATH-500 at 开放 权重, and is small enough to distill. The same paper also releases six distilled 稠密 模型 (Qwen-1.5B through Llama-70B) by SFT'ing on R1's 推理 traces — no RL at the student. Distillation of a strong RL teacher consistently beats RL from scratch at the student's 规模.

**Why GRPO instead of PPO for 推理.** Three reasons in the DeepSeekMath paper (Feb 2024): (1) no value network to 训练, halving 内存; (2) the group 基线 naturally handles the 稀疏 end-of-trajectory 奖励 that 推理 tasks produce; (3) per-prompt 归一化 makes advantages comparable across problems of wildly different difficulty, which PPO's single critic cannot.

**Search-free vs search-based.** Games have branched:

- *Perfect-information games with long horizons* (Go, chess): still search-based. AlphaZero / MuZero dominate.
- *LLM 推理*: no MCTS yet in 生产; GRPO on full rollouts, best-of-N for 推理 计算. Process 奖励模型s (PRMs) hint at step-level search being added back.

## 动手构建

这个code in `code/main.py` implements **GRPO in miniature** — a bandit with multiple groups of 样本. The algorithm is the same as on an LLM; only the 策略 and 环境 are simpler. It teaches the *损失* and the *group-relative advantage*, which is the 2025 innovation.

### 步骤 1: a tiny verifier 环境

```python
QUESTIONS = [
    {"prompt": "q1", "correct": 3},
    {"prompt": "q2", "correct": 1},
]

def verify(prompt_idx, answer_token):
    return 1.0 if answer_token == QUESTIONS[prompt_idx]["correct"] else 0.0
```

In 真实 GRPO the verifier runs unit tests or checks math equality.

### 步骤 2: 策略: softmax over K 答案 词元 per 提示词

```python
def policy_probs(theta, p_idx):
    return softmax(theta[p_idx])
```

Equivalent to the final-layer 输出 of an LLM conditioned on a 提示词.

### 步骤 3: group 采样 and group-relative advantage

```python
def grpo_step(theta, p_idx, G=8, beta=0.01, lr=0.1, rng=None):
    probs = policy_probs(theta, p_idx)
    samples = [sample(probs, rng) for _ in range(G)]
    rewards = [verify(p_idx, s) for s in samples]
    mean_r = sum(rewards) / G
    std_r = stddev(rewards) + 1e-8
    advs = [(r - mean_r) / std_r for r in rewards]

    for a, A in zip(samples, advs):
        grad = onehot(a) - probs
        for i in range(len(probs)):
            theta[p_idx][i] += lr * A * grad[i]
    # KL penalty: pull theta toward reference
    for i in range(len(probs)):
        theta[p_idx][i] -= beta * (theta[p_idx][i] - reference[p_idx][i])
```

这个group-relative advantage is the 2024 DeepSeek trick. No critic needed. The "基线" is the group mean, and 归一化 uses group std.

### 步骤 4: compare to REINFORCE 基线 (value-free)

Same setup, same 计算, plain REINFORCE. GRPO converges faster and more stably.

### 步骤 5: observe 熵 and KL

Same diagnostics as RLHF: mean KL to 参考, 策略 熵, reward-over-time. Once these stabilize, 训练 is done.

## Pitfalls

- **奖励 hacking via verifier gaming.** GRPO inherits RLHF's 风险: if the verifier is wrong or exploitable, the LLM will find the exploit. Robust verifiers (multiple test cases, formal proofs) matter.
- **Group size too small.** 方差 of the group 基线 goes like `1/√G`. Below `G = 4`, the advantage 信号 is noisy; standard choice is `G = 8` to `64`.
- **Length 偏差.** LLM completions of different lengths have different log-probabilities. Normalize by 词元 count, or use sequence-level log-prob, or truncate to max length.
- **Pure self-play cycles.** AlphaZero-style 训练 can get stuck in dominance loops on general-sum games. Mitigated by diverse opponent pools (league play, Lesson 10).
- **Search-policy mismatch.** AlphaZero trains the 策略 to mimic search 输出. If the 策略 net is too small to represent the search's 分布, 训练 stalls.
- **计算 floor.** MuZero / AlphaZero need massive 计算. A single ablation is often hundreds of GPU-hours. Miniature demos exist (e.g., AlphaZero on Connect Four) for 学习.
- **Verifier coverage.** Unit tests that pass for a buggy solution reinforce the bug. Design verifiers that catch 边 cases.

## 实际使用

这个2026 game-RL landscape, by 领域:

|领域|Dominant method|
|--------|-----------------|
|Two-player zero-sum board games (Go, chess, shogi)|AlphaZero / MuZero / KataGo|
|Imperfect info card games (poker)|CFR + deep 学习 (DeepStack, Libratus, Pluribus)|
|Atari / pixel games|Muesli / MuZero / IMPALA-PPO|
|Large multiplayer strategy (Dota, StarCraft)|PPO + self-play + league (OpenAI Five, AlphaStar)|
|LLM math/code 推理|GRPO (DeepSeek-R1, Qwen-RL, 开放 replications)|
|LLM 对齐|DPO / RLHF-PPO (not GRPO; verifier is preference not verifiable)|
|Robotics|PPO + DR (not game-RL, but uses same policy-gradient 工具)|
|Combinatorial problems|AlphaZero variants (AlphaTensor, AlphaDev)|

这个*recipe* — self-play, search-augmented improvement, 策略 distillation — spans 文本, pixels, and physical control. GRPO is the youngest instance; more are coming.

## 交付成果

Save as `outputs/skill-game-rl-designer.md`:

```markdown
---
name: game-rl-designer
description: Design a game-RL or reasoning-RL training pipeline (AlphaZero / MuZero / GRPO) for a given domain.
version: 1.0.0
phase: 9
lesson: 12
tags: [rl, alphazero, muzero, grpo, self-play]
---

Given a target (perfect-info game / imperfect-info / Atari / LLM reasoning / combinatorial), output:

1. Environment fit. Known rules? Markov? Stochastic? Multi-agent? Informs AlphaZero vs MuZero vs GRPO.
2. Search strategy. MCTS (PUCT with learned prior), Gumbel-sampled, best-of-N, or none.
3. Self-play plan. Symmetric self-play / league / offline data / verifier-generated.
4. Target signal. Game outcome / verifier reward / preference / learned model. Include robustness plan.
5. Diagnostics. Win rate vs baseline, ELO curve, verifier pass rate, KL to reference.

Refuse AlphaZero on imperfect-info games (route to CFR). Refuse GRPO without a trusted verifier. Refuse any game-RL pipeline without a fixed baseline opponent set (self-play ELO is uncalibrated otherwise).
```

## 练习

1. **Easy.** Implement the GRPO bandit in `code/main.py`. 训练 on 2 prompts × 4 答案 词元 each. Converge in < 1,000 updates with `G=8`.
2. **Medium.** Plug in PPO (clipped) and vanilla REINFORCE. Compare 样本 efficiency and 奖励 方差 to GRPO on the same bandit.
3. **Hard.** Extend to a length-2 "推理 链": the 智能体 emits two 词元 and the verifier rewards the pair. Measure how GRPO handles the credit assignment across two-step sequences. (Hint: 计算 group advantage per *full 序列*, propagate to both 词元 positions.)

## Key Terms

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|MCTS|"Tree search with learned net"|Monte Carlo Tree Search; UCB1/PUCT selection with learned `(p, v)` priors.|
|AlphaZero|"Self-play + MCTS"|Policy-value net 训练后的 to match MCTS visits and game outcome.|
|MuZero|"Learned-model AlphaZero"|Same 循环 but in 潜变量 space via learned dynamics.|
|GRPO|"Critic-free PPO"|Group Relative 策略 优化; REINFORCE with group-mean 基线 + KL.|
|PUCT|"AlphaZero's UCB"|`Q + c · p · √N / (1 + N_a)` — balances value estimate with 先验.|
|Self-play|"智能体 vs past self"|Standard for zero-sum; symmetric 训练 信号.|
|League play|"Population-based self-play"|Past + current + exploiters sampled as opponents.|
|Verifier 奖励|"Verifiable RL"|奖励 comes from a deterministic checker (tests pass, 答案 matches).|
|Process 奖励|"PRM"|Scores each 推理 步骤, not just the final 答案.|

## 延伸阅读

- [Silver et al. (2017). Mastering the game of Go without human knowledge (AlphaGo Zero)](https://www.nature.com/articles/nature24270).
- [Silver et al. (2018). A general reinforcement learning algorithm that masters chess, shogi, and Go through self-play (AlphaZero)](https://www.science.org/doi/10.1126/science.aar6404).
- [Schrittwieser et al. (2020). Mastering Atari, Go, chess and shogi by planning with a learned model (MuZero)](https://www.nature.com/articles/s41586-020-03051-4).
- [Vinyals et al. (2019). Grandmaster level in StarCraft II (AlphaStar)](https://www.nature.com/articles/s41586-019-1724-z).
- [DeepSeek-AI (2024). DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models (GRPO)](https://arxiv.org/abs/2402.03300) — the paper that introduced GRPO and the group-relative 基线.
- [DeepSeek-AI (2025). DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning](https://arxiv.org/abs/2501.12948) — the full four-stage R1 recipe plus the R1-Zero ablation.
- [Brown et al. (2019). Superhuman AI for multiplayer poker (Pluribus)](https://www.science.org/doi/10.1126/science.aay2400) — CFR + deep-learning at 规模.
- [Tesauro (1995). Temporal Difference Learning and TD-Gammon](https://dl.acm.org/doi/10.1145/203330.203343) — the paper that started it all.
- [Hugging Face TRL — GRPOTrainer](https://huggingface.co/docs/trl/main/en/grpo_trainer) — the 生产 参考 for applying GRPO with custom 奖励 函数.
- [Qwen Team (2024). Qwen2.5-Math — GRPO replication](https://github.com/QwenLM/Qwen2.5-Math) — 开放 replication of the R1 recipe at multiple scales.
- [Sutton & Barto (2018). Ch. 17 — Frontiers of Reinforcement Learning](http://incompleteideas.net/book/RLbook2020.pdf) — the textbook framing for self-play, search, and "designed 奖励" that R1 instantiates at LLM 规模.
