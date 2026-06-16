# Deep Q-Networks (DQN)

> 2013: Mnih 训练后的 one Q-learning network on raw pixels, beat every classical RL 智能体 on seven Atari games. 2015: extended to 49 games, published in Nature, sparked the deep-RL era. DQN is Q-learning plus three tricks that make 函数 approximation stable.

**类型：** Build
**语言：** Python
**先修：** Phase 3 · 03 (Backpropagation), Phase 9 · 04 (Q-learning, SARSA)
**时间：** 约 75 分钟

## 问题

Tabular Q-learning needs a separate Q-value for every (状态, 动作) pair. A chess board has ~10⁴³ states. An Atari frame is 210×160×3 = 100,800 特征s. Tabular RL dies at thousands of states, let alone billions.

这个fix is obvious in hindsight: replace the Q-table with a neural network, `Q(s, a; θ)`. But obvious-in-hindsight took decades. Naive 函数 approximation with Q-learning diverges under the "deadly triad" — 函数 approximation + bootstrapping + off-policy 学习. Mnih et al. (2013, 2015) identified three engineering tricks that stabilize 学习:

1. **Experience replay** decorrelates transitions.
2. **目标 network** freezes the bootstrap 目标.
3. **奖励 clipping** normalizes 梯度 magnitudes.

DQN on Atari was the first time a single 架构 with a single hyperparameter set solved dozens of control problems from raw pixels. Everything "deep-RL" built since — DDQN, Rainbow, Dueling, Distributional, R2D2, Agent57 — is stacked on top of this three-trick base.

## 概念

![DQN training loop: env, replay buffer, online net, target net, Bellman TD loss](../assets/dqn.svg)

**The 目标.** DQN minimizes the one-step TD 损失 on a neural Q-function:

`L(θ) = E_{(s,a,r,s')~D} [ (r + γ max_{a'} Q(s', a'; θ^-) - Q(s, a; θ))² ]`

`θ` = online network, updated every 步骤 by 梯度 descent. `θ^-` = 目标 network, periodically copied from `θ` (every ~10,000 步骤). `D` = replay buffer of past transitions.

**The three tricks, in order of importance:**

**Experience replay.** A ring buffer of `~10⁶` transitions. Each 训练 步骤 样本 a minibatch uniformly at random. This breaks temporal correlation (successive frames are nearly identical), lets the network learn from rare rewarding transitions many times, and decorrelates consecutive 梯度 updates. Without it, on-policy TD with a neural net diverges on Atari.

**目标 network.** Using the same network `Q(·; θ)` on both sides of the Bellman equation makes the 目标 move every update — "chasing your own tail." The fix: keep a second network `Q(·; θ^-)` with frozen 权重. Every `C` 步骤, copy `θ → θ^-`. This stabilizes the 回归 目标 for thousands of 梯度 步骤 at a time. Soft updates `θ^- ← τ θ + (1-τ) θ^-` (used in DDPG, SAC) are a smoother variant.

**奖励 clipping.** Atari 奖励 magnitudes vary from 1 to 1000+. Clipping to `{-1, 0, +1}` stops any single game from dominating the 梯度. Wrong when 奖励 magnitude matters; fine for Atari where only sign matters.

**Double DQN.** Hasselt (2016) fixes maximization 偏差: use the online net to *select* the 动作, the 目标 net to *evaluate* it.

`target = r + γ Q(s', argmax_{a'} Q(s', a'; θ); θ^-)`

Drop-in replacement, consistently better. Use it by default.

**Other improvements (Rainbow, 2017):** prioritized replay (样本 high-TD-error transitions more), dueling 架构 (separate `V(s)` and advantage 头), noisy networks (learned exploration), n-step returns, distributional Q (C51/QR-DQN), multi-step bootstrapping. Each adds a few percent; the gains are roughly additive.

## 动手构建

这个code here is stdlib-only numpy-free — we use a hand-rolled single-hidden-layer MLP on a tiny continuous GridWorld, so every 训练 步骤 runs in microseconds. The algorithm is identical to Atari DQN at 规模.

### 步骤 1: replay buffer

```python
class ReplayBuffer:
    def __init__(self, capacity):
        self.buf = []
        self.capacity = capacity
    def push(self, s, a, r, s_next, done):
        if len(self.buf) == self.capacity:
            self.buf.pop(0)
        self.buf.append((s, a, r, s_next, done))
    def sample(self, batch, rng):
        return rng.sample(self.buf, batch)
```

~50,000 capacity for Atari; 5,000 suffices for our toy env.

### 步骤 2: a tiny Q-network (manual MLP)

```python
class QNet:
    def __init__(self, n_in, n_hidden, n_actions, rng):
        self.W1 = [[rng.gauss(0, 0.3) for _ in range(n_in)] for _ in range(n_hidden)]
        self.b1 = [0.0] * n_hidden
        self.W2 = [[rng.gauss(0, 0.3) for _ in range(n_hidden)] for _ in range(n_actions)]
        self.b2 = [0.0] * n_actions
    def forward(self, x):
        h = [max(0.0, sum(w * xi for w, xi in zip(row, x)) + b) for row, b in zip(self.W1, self.b1)]
        q = [sum(w * hi for w, hi in zip(row, h)) + b for row, b in zip(self.W2, self.b2)]
        return q, h
```

Forward pass: linear → ReLU → linear. That is the entire net.

### 步骤 3: the DQN update

```python
def train_step(online, target, batch, gamma, lr):
    grads = zeros_like(online)
    for s, a, r, s_next, done in batch:
        q, h = online.forward(s)
        if done:
            y = r
        else:
            q_next, _ = target.forward(s_next)
            y = r + gamma * max(q_next)
        td_error = q[a] - y
        accumulate_grads(grads, online, s, h, a, td_error)
    apply_sgd(online, grads, lr / len(batch))
```

这个shape is Q-learning from Lesson 04 with two differences: (a) we backprop through a differentiable `Q(·; θ)` instead of indexing a table, (b) the 目标 uses `Q(·; θ^-)`.

### 步骤 4: the outer 循环

For each episode, act ε-greedy on `Q(·; θ)`, push transitions into the buffer, 样本 a minibatch, take a 梯度 步骤, periodically sync `θ^- ← θ`. The pattern:

```python
for episode in range(N):
    s = env.reset()
    while not done:
        a = epsilon_greedy(online, s, epsilon)
        s_next, r, done = env.step(s, a)
        buffer.push(s, a, r, s_next, done)
        if len(buffer) >= batch:
            train_step(online, target, buffer.sample(batch), gamma, lr)
        if steps % sync_every == 0:
            target = copy(online)
        s = s_next
```

On our tiny GridWorld with a 16-dim one-hot 状态, the 智能体 learns a near-optimal 策略 in ~500 episodes. On Atari, 规模 this to 200M frames and add a CNN 特征 extractor.

## Pitfalls

- **Deadly triad.** 函数 approximation + off-policy + bootstrapping can diverge. DQN mitigates with 目标 net + replay; do not remove either.
- **Exploration.** ε must 衰减, typically from 1.0 to 0.01 over the first ~10% of 训练. Without enough early exploration the Q-net converges to a local basin.
- **Overestimation.** `max` over noisy Q is upward-biased. Always use Double DQN in 生产.
- **奖励 规模.** Clip or normalize rewards; the 梯度 magnitude is proportional to 奖励 magnitude.
- **Replay buffer coldstart.** Don't 训练 until the buffer has a few thousand transitions. Early gradients on ~20 样本 overfit.
- **目标 sync frequency.** Too frequent ≈ no 目标 net; too infrequent ≈ stale 目标. Atari DQN uses 10,000 env 步骤. Rule of thumb: sync every ~1/100 of 训练 horizon.
- **Observation preprocessing.** Atari DQN stacks 4 frames to make 状态 Markov. Any env with velocity info needs frame-stacking or recurrent 状态.

## 实际使用

In 2026, DQN is rarely state-of-the-art but remains the 参考 off-policy algorithm:

|任务|Method of choice|Why not DQN?|
|------|------------------|--------------|
|Discrete-action Atari-like|Rainbow DQN or Muesli|Same framework, more tricks.|
|Continuous control|SAC / TD3 (Phase 9 · 07)|DQN has no 策略 network.|
|On-policy / high-throughput|PPO (Phase 9 · 08)|No replay buffer; easier to 规模.|
|Offline RL|CQL / IQL / Decision Transformer|Conservative Q 目标, no bootstrapping blowups.|
|Large discrete 动作 spaces (recommender)|DQN with 动作 嵌入, or IMPALA|Fine; decoration matters.|
|LLM RL|PPO / GRPO|Sequence-level, not step-level; different 损失.|

这个lessons still travel. Replay and 目标 networks appear in SAC, TD3, DDPG, SAC-X, AlphaZero's self-play buffer, and every offline RL method. 奖励 clipping lives on as advantage 归一化 in PPO. The 架构 is the blueprint.

## 交付成果

Save as `outputs/skill-dqn-trainer.md`:

```markdown
---
name: dqn-trainer
description: Produce a DQN training config (buffer, target sync, ε schedule, reward clipping) for a discrete-action RL task.
version: 1.0.0
phase: 9
lesson: 5
tags: [rl, dqn, deep-rl]
---

Given a discrete-action environment (observation shape, action count, horizon, reward scale), output:

1. Network. Architecture (MLP / CNN / Transformer), feature dim, depth.
2. Replay buffer. Capacity, minibatch size, warmup size.
3. Target network. Sync strategy (hard every C steps or soft τ).
4. Exploration. ε start / end / schedule length.
5. Loss. Huber vs MSE, gradient clip value, reward clipping rule.
6. Double DQN. On by default unless explicit reason to disable.

Refuse to ship a DQN with no target network, no replay buffer, or ε held at 1. Refuse continuous-action tasks (route to SAC / TD3). Flag any reward range > 10× per-step mean as needing clipping or scale normalization.
```

## 练习

1. **Easy.** Run `code/main.py`. Plot the per-episode return 曲线. How many episodes until the running mean exceeds -10?
2. **Medium.** Disable the 目标 network (use the online net for both sides of the Bellman 目标). Measure 训练 instability — does return oscillate or diverge?
3. **Hard.** Add Double DQN: use the online net to pick `argmax a'`, 目标 net to evaluate. Compare 偏差 of `Q(s_0, best_a)` vs true `V*(s_0)` after 1,000 episodes with vs without Double DQN on a noisy-reward GridWorld.

## Key Terms

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|DQN|"Deep Q-learning"|Q-learning with a neural Q-function, replay buffer, and 目标 network.|
|Experience replay|"Shuffled transitions"|Ring buffer sampled uniformly each 梯度 步骤; decorrelates 数据.|
|目标 network|"Frozen bootstrap"|Periodic copy of Q used in the Bellman 目标; stabilizes 训练.|
|Deadly triad|"Why RL diverges"|函数 approximation + bootstrapping + off-policy = no convergence guarantee.|
|Double DQN|"修复 for maximization 偏差"|Online net selects 动作, 目标 net evaluates it.|
|Dueling DQN|"V and A 头"|Decompose Q = V + A - mean(A); same 输出, better 梯度 流.|
|Rainbow|"All the tricks"|DDQN + PER + dueling + n-step + noisy + distributional in one.|
|PER|"Prioritized Replay"|样本 transitions proportional to TD-error magnitude.|

## 延伸阅读

- [Mnih et al. (2013). Playing Atari with Deep Reinforcement Learning](https://arxiv.org/abs/1312.5602) — the 2013 NeurIPS workshop paper that kicked off deep RL.
- [Mnih et al. (2015). Human-level control through deep reinforcement learning](https://www.nature.com/articles/nature14236) — the Nature paper, 49-game DQN.
- [Hasselt, Guez, Silver (2016). Deep Reinforcement Learning with Double Q-learning](https://arxiv.org/abs/1509.06461) — DDQN.
- [Wang et al. (2016). Dueling Network Architectures](https://arxiv.org/abs/1511.06581) — dueling DQN.
- [Hessel et al. (2018). Rainbow: Combining Improvements in Deep RL](https://arxiv.org/abs/1710.02298) — the stacked-tricks paper.
- [OpenAI Spinning Up — DQN](https://spinningup.openai.com/en/latest/algorithms/dqn.html) — clear modern exposition.
- [Sutton & Barto (2018). Ch. 9 — On-policy Prediction with Approximation](http://incompleteideas.net/book/RLbook2020.pdf) — the textbook treatment of the "deadly triad" (函数 approximation + bootstrapping + off-policy) that DQN's 目标 network and replay buffer are designed to tame.
- [CleanRL DQN implementation](https://docs.cleanrl.dev/rl-algorithms/dqn/) — 参考 single-file DQN used in ablation studies; good to read alongside this lesson's from-scratch version.
