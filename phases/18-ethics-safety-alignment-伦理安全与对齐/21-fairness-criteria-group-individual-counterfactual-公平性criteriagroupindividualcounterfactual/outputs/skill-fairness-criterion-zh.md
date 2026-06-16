---
name: fairness-criterion-zh
description: Identify which 公平性 criterion a claim invokes and 审计 the associated assumptions.
version: 1.0.0
phase: 18
lesson: 21
tags: [fairness, demographic-parity, equalized-odds, counterfactual-fairness, impossibility]
---

给定 a 公平性 claim or 策略, identify which criterion is being invoked, what assumptions the claim depends on, and what the impossibility theorems imply for the remaining criteria.

产出：

1. Criterion identification. Label the claim as targeting one of: demographic parity, equalized odds, conditional use accuracy equality, individual 公平性, counterfactual 公平性. Ambiguous claims must be resolved before proceeding.
2. Base-rate 审计. What are the per-group base rates in the 部署? Under unequal base rates, Chouldechova / KMR 2017 impossibility applies: no 模型 satisfies all three group criteria.
3. Causal-DAG dependency. If the claim is counterfactual 公平性, what is the causal DAG? Counterfactual 公平性 is only as justified as the DAG. Lack of a DAG invalidates the claim.
4. Similarity metric. If the claim is individual 公平性, what is the similarity metric d? The choice is task-specific and is a 策略 decision, not a statistical one.
5. Intervention legality. If the claim uses counterfactual reasoning, are interventions on protected attributes involved? If yes, consider backtracking counterfactuals (arXiv:2401.13935) to sidestep legal issues.

硬性拒绝：
- Any "fair" claim without criterion identification.
- Any "all 公平性 criteria satisfied" claim under unequal base rates without acknowledging Chouldechova / KMR 2017.
- Any counterfactual-公平性 claim without a published causal DAG.

拒绝规则：
- 如果用户询问 which 公平性 criterion is "the right one," refuse the ranking and explain it is a 策略 choice.
- 如果用户询问 whether a 模型 is "fair," refuse the binary claim; 公平性 is criterion-relative.

输出：a one-page 审计 filling the five sections above, flagging the impossibility if applicable, and naming the 策略 choice implicit in the claim. 引用 Dwork et al. 2012, Kusner et al. 2017, Chouldechova 2017 once each as appropriate.
