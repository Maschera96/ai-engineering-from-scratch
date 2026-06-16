# 公平性 Criteria ， Group, Individual, Counterfactual

> Three families structure the 公平性 literature. Group 公平性: demographic parity, equalized odds, conditional use accuracy equality ， equal rates across protected groups on average. Individual 公平性 (Dwork et al. 2012): similar individuals receive similar decisions; Lipschitz condition on the decision map. Counterfactual 公平性 (Kusner et al. 2017): a decision is fair to an individual if it is unchanged when sensitive attributes are counterfactually altered. 2024 theoretical result (NeurIPS 2024): there is an inherent CF-vs-accuracy trade-off; a 模型-agnostic method converts an optimal-but-unfair predictor into a CF one with bounded accuracy loss. Backtracking counterfactuals (arXiv:2401.13935, January 2024): new paradigm that avoids requiring interventions on legally protected attributes. Philosophical reconciliation (ICLR Blogposts 2024): with causal graphs, satisfying certain group 公平性 measures entails counterfactual 公平性.

**类型:** 学习
**语言:** Python (stdlib, three-criteria comparison)
**先修要求:** Phase 18 · 20 (偏差), Phase 02 (classical ML)
**时间:** ~60 分钟

## 学习目标

- 说明 the three group-公平性 criteria (demographic parity, equalized odds, conditional use accuracy equality) and one impossibility result.
- 描述 individual 公平性 via the Dwork et al. 2012 Lipschitz formulation.
- 描述 counterfactual 公平性 and its causal-graph dependency.
- 解释 backtracking counterfactuals and why they sidestep the intervention-on-protected-attribute problem.

## 问题

Lesson 20 was about measuring 偏差. Lesson 21 is about defining the 公平性 standard the measurement should serve. The three families give structurally different standards ， a 模型 can be group-fair and individual-unfair, counterfactually fair and group-unfair. Choosing a standard is a 策略 decision; no standard is universally optimal.

## 概念

### Group 公平性

- **Demographic parity.** P(Y=1 | A=a) = P(Y=1 | A=a') for all groups. Equal acceptance rates.
- **Equalized odds.** P(Y=1 | Y*=y, A=a) = P(Y=1 | Y*=y, A=a'). Equal TPR and FPR across groups.
- **Conditional use accuracy equality.** P(Y*=y | Y=y, A=a) = P(Y*=y | Y=y, A=a'). Equal predictive value across groups.

Impossibility (Chouldechova, Kleinberg-Mullainathan-Raghavan 2017): these three cannot be satisfied simultaneously under unequal base rates.

### Individual 公平性

Dwork et al. 2012. A decision map f is individually fair with respect to a task-specific similarity metric d if |f(x) - f(x')| <= L * d(x, x') for some Lipschitz constant L. Similar individuals get similar decisions.

Requires defining d. 策略 question, not statistical.

### Counterfactual 公平性

Kusner et al. 2017. A decision is counterfactually fair to individual i if, under a causal 模型 of the population, the decision is unchanged when i's sensitive attributes are counterfactually altered.

Requires a causal DAG. The DAG is a modeling choice. Counterfactual 公平性 is only as justified as the DAG.

### The CF-vs-accuracy trade-off

NeurIPS 2024 theoretical: there is an inherent trade-off between counterfactual 公平性 and predictive accuracy. A 模型-agnostic method can convert an optimal-but-unfair predictor into a CF one, at a bounded accuracy cost. The accuracy cost depends on the magnitude of the sensitive-attribute coefficient in the optimal unfair predictor.

### Backtracking counterfactuals

arXiv:2401.13935 (January 2024). Traditional counterfactuals require interventions on the sensitive attribute ， "would the decision change if this person had been a different gender." Legally, this is problematic: protected attributes cannot be intervened on in classification law.

Backtracking counterfactuals flip the direction: instead of intervening on the attribute, ask what combination of the individual's actual features would have produced the counterfactual outcome. This sidesteps the legal objection.

### Philosophical reconciliation

ICLR Blogposts 2024. With a causal graph in hand, satisfying certain group-公平性 measures entails counterfactual 公平性. The three families are not orthogonal; they are different facets of the same underlying causal structure.

This does not resolve the impossibility theorems (unequal base rates still prevent simultaneous group 公平性). But it shows the apparent opposition between "group" and "individual / counterfactual" is partially an artifact of not being explicit about the causal 模型.

### Where this fits in Phase 18

Lesson 20 is 偏差 measurement. Lesson 21 is 公平性 definition. Lesson 22 is 隐私 (差分隐私). Lesson 23 is 水印. These are the allocation-adjacent lessons complementing the deception-adjacent Lessons 7-11.

## 使用它

`code/main.py` builds a toy binary-classification 数据集 with a sensitive attribute and unequal base rates. Compute demographic parity, equalized odds, and conditional use accuracy equality on a simple 分类器. Observe the three metrics disagreeing. Apply a re-weighting for demographic parity and observe its cost on the other two.

## 交付它

本课产出 `outputs/skill-fairness-criterion.md`. 给定 a 公平性 claim or 策略, identifies which criterion is being claimed, whether the 模型 can satisfy the remaining criteria under the claimed unequal base rates, and what causal DAG the claim depends on.

## 练习

1. 运行 `code/main.py`. Report the three group metrics on the default data. Apply the demographic-parity-targeted re-weighting and re-report.

2. Implement the Dwork et al. 2012 individual-公平性 metric using L2 on non-sensitive features. Report how many pairs violate Lipschitz with constant L=1.

3. 阅读 Kusner et al. 2017. Construct a simple two-feature causal DAG for resume scoring and identify the counterfactual-公平性 condition it implies.

4. The 2024 backtracking-counterfactuals paper avoids intervention on protected attributes. 描述 a scenario where this matters for legal 服从.

5. The ICLR 2024 reconciliation argues group and counterfactual 公平性 are facets of the same structure. Pick two of the three criteria in `code/main.py` and state the causal assumption that would make them equivalent.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|-----------------|------------------------|
| Demographic parity | "equal rates" | P(Y=1 | A=a) equal across groups |
| Equalized odds | "equal TPR/FPR" | Equal true-positive and false-positive rates across groups |
| Conditional use accuracy | "equal PPV/NPV" | Equal predictive values across groups |
| Individual 公平性 | "Lipschitz condition" | Similar individuals get similar decisions |
| Counterfactual 公平性 | "causal alteration invariance" | Decision unchanged under counterfactual attribute alteration |
| Backtracking counterfactual | "explain via actuals" | Counterfactual reasoned backward from outcome, not forward from attribute |
| Impossibility theorem | "the three conflict" | Chouldechova / KMR 2017: group criteria mutually exclusive under unequal base rates |

## 延伸阅读

- [Dwork et al. ， Fairness through Awareness (arXiv:1104.3913)](https://arxiv.org/abs/1104.3913) ， individual 公平性
- [Kusner, Loftus, Russell, Silva ， Counterfactual Fairness (arXiv:1703.06856)](https://arxiv.org/abs/1703.06856) ， counterfactual 公平性
- [Chouldechova ， Fair prediction with disparate impact (arXiv:1703.00056)](https://arxiv.org/abs/1703.00056) ， impossibility
- [Backtracking Counterfactuals (arXiv:2401.13935)](https://arxiv.org/abs/2401.13935) ， new paradigm for protected-attribute interventions
