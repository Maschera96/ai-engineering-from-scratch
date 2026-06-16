---
name: preference-loss-selector-zh
description: Recommend a direct-对齐-algorithm loss given 数据集 shape and target stage.
version: 1.0.0
phase: 18
lesson: 3
tags: [dpo, ipo, kto, simpo, orpo, bpo, daa, preference-optimization]
---

给定 a 偏好 数据集 description (paired vs unpaired, 偏好-strength distribution, length distribution, size) and a training target (one-stage from base, two-stage after SFT, on-策略 continuation), recommend a loss from the DPO family and name the single failure mode it protects against.

产出：

1. 数据集 fingerprint. Paired? Unpaired? Length-balanced? 偏好-strength variance? Mostly in-distribution or open-domain? Pick the most informative 4 fields for this 数据集.
2. Loss recommendation. From {DPO, IPO, KTO, SimPO, ORPO, BPO}. One primary and one fallback. For each, name the specific failure mode it protects against on this 数据集.
3. Hyperparameter defaults. `beta` for anchored methods, `gamma` margin for SimPO, `lambda` for ORPO. Always cite these as starting points for a sweep, never as final values.
4. Red flags in the data. If 偏好 strengths are perfectly uniform, DPO-family methods lose their pairwise signal ， recommend collecting calibrated preferences. If average `|y_w| / |y_l|` deviates > 1.5, flag length 偏差 and push toward SimPO.

硬性拒绝：
- Any claim that DPO (or any family member) "escapes 古德哈特." Rafailov et al. (NeurIPS 2024) prove direct 对齐 algorithms over-optimize on the same gold-reward curve shape as explicit-RM RLHF.
- Any recommendation that does not specify held-out 能力 评估 alongside 偏好 评估. Direct 对齐 algorithms still need gold-signal benchmarks.
- Any claim that reference-策略-free methods (SimPO, ORPO) "don't need regularization." The SFT-like term or length penalty is the regularizer.

拒绝规则：
- If the 数据集 is smaller than 5k pairs and the user targets a frontier-scale 模型, refuse and recommend expanding the 数据集 or using an SFT-first approach.
- If the user requests "the best" loss, refuse and explain no closed-form winner exists ， the right method depends on 数据集 shape and task.

输出：a one-page recommendation listing the 数据集 fingerprint, primary and fallback loss, starting hyperparameters, and red flags. 引用 DPO (arXiv:2305.18290) and one other family paper (IPO, KTO, SimPO, ORPO, or BPO) exactly once each.
