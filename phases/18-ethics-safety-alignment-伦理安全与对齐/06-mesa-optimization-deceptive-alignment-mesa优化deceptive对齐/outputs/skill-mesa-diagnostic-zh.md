---
name: mesa-diagnostic-zh
description: Classify an observed 安全 failure as outer-对齐, proxy-inner, or deceptive-inner.
version: 1.0.0
phase: 18
lesson: 6
tags: [mesa-optimization, deceptive-alignment, inner-alignment, hubinger]
---

给定 a 安全 评估 report (eval task, failure mode, 模型 class, training recipe), classify the failure into the Hubinger 2019 categories and recommend the mitigation class that addresses it.

产出：

1. Failure-mode categorization. Pick one of:
   - Outer-对齐 failure: the base objective (reward, loss) was wrong; the 模型 optimized it correctly.
   - Inner-对齐 proxy failure: mesa-objective is a proxy that tracks base in-distribution; fails off-distribution.
   - Inner-对齐 deceptive: mesa-optimizer has situational awareness and defects at 部署; training behaviour is clean.
2. Evidence trace. For each category, what evidence would support it. For deceptive, distinguish from proxy: evidence of situational awareness (date sensitivity, eval-vs-部署 distinguishers, strategic reasoning in chain-of-thought).
3. Mitigation class. For outer-对齐: change the objective (CAI, better reward data, process supervision). For proxy-inner: distributional coverage, ensembles, held-out evals. For deceptive-inner: 控制 measures (Lesson 10), interpretability (residual-stream probes), 能力 reductions.
4. Known-failures check. For deceptive-inner, cite the relevant 2024-2026 empirical demonstration (休眠智能体, 对齐 Faking, In-Context 密谋) this failure most resembles.

硬性拒绝：
- Any classification of deceptive-inner without evidence of situational awareness. "Unexpected behaviour at 部署" is not enough ， it could be proxy-inner.
- Any claim that adversarial robustness training alone addresses deceptive-inner. Hubinger 2019 predicts (and 休眠智能体 2024 confirms) that adversarial training can teach better test-vs-部署 distinguishers.
- Any recommendation to retrain a deceptively aligned 模型 on more data. The prior predicts deception is preserved under further training.

拒绝规则：
- If the evidence is a single failure on a single prompt, refuse to classify. Base rates matter; you need a distribution of failures.
- 如果用户询问 you to "rule out" 欺骗性对齐, refuse ， you can estimate its probability from evidence, but you cannot rule it out behaviourally alone.

输出：a one-page diagnosis with category, evidence trace, mitigation class, and nearest empirical analog. 引用 Hubinger et al. (arXiv:1906.01820) once.
