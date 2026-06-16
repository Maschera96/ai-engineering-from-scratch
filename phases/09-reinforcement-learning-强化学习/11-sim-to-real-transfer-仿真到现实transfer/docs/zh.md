# Sim-to-Real Transfer

> 一个策略 训练后的 in a simulator that fails on hardware is a 策略 that memorized the simulator. 领域 randomization, 领域 adaptation, and 系统 identification are the three 工具 to make learned controllers cross the reality gap.

**类型：** Learn
**语言：** Python
**先修：** Phase 9 · 08 (PPO), Phase 2 · 10 (偏差/方差)
**时间：** 约 45 分钟

## 问题

训练 a 真实 robot is slow, dangerous, and expensive. A biped takes millions of 训练 episodes to learn to walk; a 真实 biped that falls over even once breaks hardware. Simulation gives you unlimited resets, deterministic reproducibility, 并行 environments, and no physical damage.

But simulators are wrong. Bearings have more friction than MuJoCo 模型. Cameras have lens distortion the simulator does not include. Motors have delays, backlash, and saturation that 99% of sim 模型 skip. Wind, dust, and variable lighting sabotage a 策略 训练后的 on sterile rendering. The **reality gap** — systematic difference between sim 分布 and 真实 分布 — is the central problem of deployed RL for robotics.

你need a 策略 that is *robust to sim-to-real 分布 shift*. Three historical approaches: randomize the simulator (领域 randomization), adapt the 策略 with a little 真实 数据 (领域 adaptation / 微调), or identify the 真实 系统's 参数 and match them (系统 identification). In 2026 the dominant recipe combines all three with massive 并行 simulation (Isaac Sim, Isaac Lab, Mujoco MJX on GPU).

## 概念

![Three sim-to-real regimes: domain randomization, adaptation, system identification](../assets/sim-to-real.svg)

**领域 Randomization (DR).** Tobin et al. 2017, Peng et al. 2018. During 训练, randomize every sim 参数 that might differ on the 真实 robot: masses, friction coefficients, motor PD gains, sensor 噪声, camera position, lighting, textures, contact 模型. The 策略 learns a 条件式 分布 over "which sim it is in today" and generalizes across the full span. If the 真实 robot falls within the 训练 envelope, the 策略 works.

- **Upside:** no 真实 数据 needed. One recipe, many robots.
- **Downside:** over-randomized 训练 produces a "universal" but overly cautious 策略. Too much 噪声 ≈ too much 正则化.

**系统 Identification (SI).** Fit the simulator's 参数 to real-world 数据 before 训练. If you can measure arm-joint friction on the 真实 robot, plug that into the sim. Then 训练 a 策略 that expects those values. Needs access to the 真实 系统 but reduces the reality gap directly.

- **Upside:** precise, low-noise 训练 目标.
- **Downside:** residual 模型 错误 is invisible to the 策略; small un-identified effects (e.g., motor deadband) still break deployment.

**领域 Adaptation.** 训练 in sim, fine-tune with a small amount of 真实 数据. Two flavors:

- **Real2Sim2Real:** learn a residual simulator `f(s, a, z) - f_sim(s, a)` using 真实 rollouts, 训练 in the corrected sim. Closes the gap without much 真实 数据.
- **Observation adaptation:** 训练 a 策略 that maps 真实 obs → sim-like obs via a learned 特征 extractor (e.g., GAN pixel-to-pixel). The controller stays in sim.

**Privileged 学习 / teacher-student.** Miki et al. 2022 (ANYmal quadruped). 训练 a *teacher* in simulation that has access to privileged information (ground truth friction, terrain height, IMU drift). Distill a *student* that only sees real-sensor observations. The student learns to infer privileged 特征s from history, robust across physical 参数.

**Massively 并行 simulation.** 2024–2026. Isaac Lab, Mujoco MJX, Brax all run thousands of 并行 robots on a single GPU. PPO with 4,096 并行 humanoids collects years of experience in 小时. The "reality gap" shrinks as 训练 分布 widens; DR becomes almost free when each of those 4,096 envs has different randomized 参数.

**The real-world 2026 recipe (quadruped walking example):**

1. Massively 并行 sim with domain-randomized gravity, friction, motor gains, payload.
2. Teacher 策略 训练后的 with privileged info (terrain map, body velocity ground truth).
3. Student 策略 distilled from teacher using only proprioception (leg joint encoders).
4. Optional observation adaptation via 自编码器 on 真实 IMU.
5. Deploy. 零样本 on 10+ environments. If it fails, do 分钟 of real-world 微调 with safety-constrained PPO.

## 动手构建

这lesson's code is a tiny demonstration of 领域 randomization on a GridWorld with *noisy* transitions. We 训练 a 策略 that experiences randomized slip 概率 in "sim" and evaluate on "真实" with a slip level it never saw during 训练. The shape maps directly to MuJoCo-to-hardware transfer.

### 步骤 1: parameterized sim

```python
def step(state, action, slip):
    if rng.random() < slip:
        action = random_perpendicular(action)
    ...
```

`slip` is a 参数 the simulator exposes. In 真实 robotics it could be friction, mass, motor gain — anything that shifts between sim and 真实.

### 步骤 2: 训练 with DR

At the start of each episode, 样本 `slip ~ Uniform[0.0, 0.4]`. 训练 PPO / Q-learning / anything. Do this for many episodes.

### 步骤 3: evaluate 零样本 on "真实" slips

Evaluate on `slip ∈ {0.0, 0.1, 0.2, 0.3, 0.5, 0.7}`. The first four are within 训练 support; `0.5` and `0.7` are outside. A DR-trained 策略 should stay near-optimal inside support and degrade gracefully outside. A fixed-slip-trained 策略 will be brittle outside its 训练 slip.

### 步骤 4: compare to narrow 训练

训练 a second 策略 with `slip = 0.0` only. Evaluate on the same `slip` sweep. You should see a catastrophic drop as soon as 真实 slip > 0.

## Pitfalls

- **Too much randomization.** 训练 on `slip ∈ [0, 0.9]` and your 策略 is so risk-averse it never tries the optimal path. Match the *expected* real-world 分布, not "anything could happen."
- **Too little randomization.** 训练 on a thin slice and the 策略 can't generalize at all. Use adaptive curriculum (Automatic 领域 Randomization) that widens the 分布 as the 策略 improves.
- **Misidentified 参数 space.** Randomize the wrong thing (camera hue when the 真实 gap is motor delay) and DR does not help. Profile the 真实 robot first.
- **Privileged info leakage.** A teacher that uses global 状态 for actions, not just observations, can produce a student that cannot catch up. Ensure the teacher's 策略 is realizable by the student given observation history.
- **Sim-to-sim transfer failure.** If your 策略 is not robust to a harder sim variant, it will not be robust to the 真实 world either. Always test on a held-out sim variant before deploying.
- **No real-world 安全 envelope.** A 策略 that works in sim and "works in 真实" without a low-level 安全 shield can still break hardware. Add 速率 limits, torque limits, joint limits in a non-learned controller.

## 实际使用

这个2026 sim-to-real stack:

|领域|Stack|
|--------|-------|
|Legged locomotion (ANYmal, Spot, humanoid)|Isaac Lab + DR + privileged teacher / student|
|Manipulation (dexterous hands, pick-and-place)|Isaac Lab + DR + DR-GAN for vision|
|Autonomous driving|CARLA / NVIDIA DRIVE Sim + DR + 真实 fine-tune|
|Drone racing|RotorS / Flightmare + DR + online adaptation|
|Finger/in-hand manipulation|OpenAI Dactyl (DR at unprecedented 规模)|
|Industrial arms|MuJoCo-Warp + SI + small 真实 fine-tune|

For control at all scales, the 工作流 is consistent: fit the sim as best you can, randomize what you can't fit, 训练 enormous policies, distill, deploy with a 安全 shield.

## 交付成果

Save as `outputs/skill-sim2real-planner.md`:

```markdown
---
name: sim2real-planner
description: Plan a sim-to-real transfer pipeline for a given robot + task, covering DR, SI, and safety.
version: 1.0.0
phase: 9
lesson: 11
tags: [rl, sim2real, robotics, domain-randomization]
---

Given a robot platform, a task, and access to real hardware time, output:

1. Reality gap inventory. Suspected sources ranked by expected impact (contact, sensing, actuation delay, vision).
2. DR parameters. Exact list, ranges, distribution. Justify each range against real measurements.
3. SI steps. Which parameters to measure; measurement method.
4. Teacher/student split. What privileged info the teacher uses; what obs the student uses.
5. Safety envelope. Low-level limits, emergency stops, backup controller.

Refuse to deploy without (a) a zero-shot sim-variant test, (b) a safety shield, (c) a rollback plan. Flag any DR range wider than 3× measured real variability as likely over-randomized.
```

## 练习

1. **Easy.** 训练 a Q-learning 智能体 on the fixed-slip GridWorld (slip=0.0). Evaluate on slip ∈ {0.0, 0.1, 0.3, 0.5}. Plot return vs slip.
2. **Medium.** 训练 a DR Q-learning 智能体 采样 `slip ~ Uniform[0, 0.3]`. Evaluate the same sweep. How much does DR buy at slip=0.5 (out-of-distribution)?
3. **Hard.** Implement a curriculum: start with slip=0.0, widen the DR range every time the 策略 hits 90% of optimal. Measure total 环境 步骤 to reach slip=0.3 零样本 vs. a fixed DR 基线.

## Key Terms

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|Reality gap|"Sim-to-real difference"|分布 shift between 训练 and deployment physics/sensing.|
|领域 randomization (DR)|"训练 across random sims"|Randomize sim 参数 during 训练 so 策略 generalizes.|
|系统 identification (SI)|"Measure 真实 and fit sim"|Estimate 真实 physical 参数; set sim to match.|
|领域 adaptation|"Fine-tune on 真实 数据"|Small real-world fine-tune after sim 训练; may adapt obs or dynamics.|
|Privileged info|"Ground truth for teacher"|Information only the sim has; student must infer it from obs history.|
|Teacher/student|"Distill privileged -> observable"|Teacher 训练后的 with shortcuts; student learns to mimic without them.|
|ADR|"Automatic 领域 Randomization"|Curriculum that widens DR ranges as the 策略 improves.|
|Real2Sim|"Close the gap with 真实 数据"|Learn a residual to make the sim mimic 真实 rollouts.|

## 延伸阅读

- [Tobin et al. (2017). Domain Randomization for Transferring Deep Neural Networks from Simulation to the Real World](https://arxiv.org/abs/1703.06907) — the original DR paper (vision for robotics).
- [Peng et al. (2018). Sim-to-Real Transfer of Robotic Control with Dynamics Randomization](https://arxiv.org/abs/1710.06537) — DR for dynamics, quadruped locomotion.
- [OpenAI et al. (2019). Solving Rubik's Cube with a Robot Hand](https://arxiv.org/abs/1910.07113) — Dactyl, ADR at 规模.
- [Miki et al. (2022). Learning robust perceptive locomotion for quadrupedal robots in the wild](https://www.science.org/doi/10.1126/scirobotics.abk2822) — teacher-student for ANYmal.
- [Makoviychuk et al. (2021). Isaac Gym: High Performance GPU Based Physics Simulation for Robot Learning](https://arxiv.org/abs/2108.10470) — the massively 并行 sim that drives 2025–2026 deployments.
- [Akkaya et al. (2019). Automatic Domain Randomization](https://arxiv.org/abs/1910.07113) — ADR curriculum method.
- [Sutton & Barto (2018). Ch. 8 — Planning and Learning with Tabular Methods](http://incompleteideas.net/book/RLbook2020.pdf) — the Dyna framing (use a 模型 for planning + rollouts) that underpins modern sim-to-real pipelines.
- [Zhao, Queralta & Westerlund (2020). Sim-to-Real Transfer in Deep Reinforcement Learning for Robotics: a Survey](https://arxiv.org/abs/2009.13303) — taxonomy of sim-to-real methods with 基准 results.
