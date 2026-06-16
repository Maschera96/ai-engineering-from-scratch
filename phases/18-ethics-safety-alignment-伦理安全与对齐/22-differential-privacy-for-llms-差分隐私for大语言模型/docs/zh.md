# 面向大语言模型的差分隐私

> DP-SGD remains the standard ， noise-injected gradient updates provide formal (epsilon, delta) guarantees. Overhead in compute, memory, and utility is substantial; parameter-efficient DP 微调 (LoRA + DP-SGD) is the common 2025 configuration (ACM 2025). Two bodies of evidence in tension: canary-based membership inference (Duan et al., 2024) reports limited success against language models; training-data extraction (Carlini et al., 2021; Nasr et al., 2025) recovers substantial verbatim memorization. Resolution (arXiv:2503.06808, March 2025): the gap is in what is measured ， inserted canaries vs "most extractable" data. New canary designs enable loss-based MIA without shadow models and yield the first nontrivial DP 审计 of an LLM trained on real data with realistic DP guarantees. Alternatives: PMixED (arXiv:2403.15638) ， private prediction at inference time via mixture of experts on next-token distributions; DP synthetic data generation (Google Research 2024). Emerging 攻击: 差分隐私 Reversal via LLM Feedback ， confidence-score leakage.

**类型:** 构建
**语言:** Python (stdlib, DP-SGD noise-injection and ε-δ accountant demonstration)
**先修要求:** Phase 01 · 09 (information theory), Phase 10 · 01 (large-模型 training)
**时间:** ~60 分钟

## 学习目标

- Define (epsilon, delta)-差分隐私 and state the DP-SGD recipe.
- 解释 the 2024-2025 tension: canary MIA vs training-data extraction give different pictures.
- 描述 PMixED and why inference-time private prediction is an alternative to DP training.
- 描述 the 差分隐私 Reversal via LLM Feedback 攻击.

## 问题

LLMs memorize. Carlini et al. 2021 showed production language models reproduce verbatim training text on demand. DP is the formal 防御: train so that the 输出 is provably insensitive to any single training example. The 2024-2025 evidence shows DP-SGD is necessary but the deployed ε values may not match the threat 模型.

## 概念

### (ε, δ)-差分隐私

A randomized algorithm M is (ε, δ)-DP if for any two datasets differing in one example and any event S:
P(M(D) in S) <= e^ε * P(M(D') in S) + δ.

Interpretation: the 输出 distribution is close enough (parametrized by ε) that the contribution of any single individual cannot be reliably inferred, except with probability δ.

### DP-SGD

Abadi et al. 2016. The standard recipe:
1. Sample a mini-batch.
2. Compute per-example gradients.
3. Clip each per-example gradient to a threshold C.
4. Sum the clipped gradients and add Gaussian noise with std σ * C.
5. Use the noisy sum to update parameters.

隐私 cost is tracked by an accountant (Moments Accountant, Rényi DP accountant). Reported ε values in the LLM literature vary widely by threat 模型, data sensitivity, and utility target; there is no universally "safe" default ε. Published examples span roughly ε ≈ 1-10 in some LLM training settings, but these are illustrative ， not recommended defaults. Lower ε generally requires more noise and can increase utility loss.

### LoRA + DP-SGD

Full DP-SGD of a frontier 模型 is prohibitive. LoRA (Hu et al. 2022) limits gradient updates to a small adapter, reducing per-example gradient storage. LoRA + DP-SGD is the common 2025 configuration. DP guarantees apply to the adapter; the base 模型 is held fixed.

### The 2024-2025 tension

Two lines of evidence:

- **Canary MIA (Duan et al. 2024).** Insert unique canaries into training data, measure whether a membership-inference attacker can identify them. Reports limited success on language models. Suggests MIA is hard.
- **Training-data extraction (Carlini 2021, Nasr et al. 2025).** Prompt the 模型 with a prefix; measure whether it recovers verbatim text from training. Reports substantial memorization. Suggests MIA is easy in the relevant sense.

March 2025 resolution (arXiv:2503.06808): the two measure different things. MIA asks "is example e in D?" on inserted canaries. Extraction asks "what can I recover of D?" The "most extractable" example is what matters for 隐私; canaries under-report this because they are not optimized to be extractable.

New canary designs. Loss-based MIA without shadow models. First nontrivial DP 审计 of an LLM on real data with realistic DP guarantees.

### Alternatives to DP training

- **PMixED (arXiv:2403.15638).** Private prediction at inference time. Mixture of experts on next-token distributions; each expert sees a shard of training data; aggregation adds noise for DP. Avoids DP training entirely.
- **DP synthetic data generation (Google Research 2024).** LoRA-微调 with DP-SGD, sample synthetic data, train a downstream 分类器 on the synthetic data.

Both sidestep the utility cost of full DP training at the cost of a different threat 模型.

### 差分隐私 Reversal via LLM Feedback

Emerging 2025 攻击. Use a DP-trained 模型's confidence scores as an oracle to re-identify individuals. Even when outputs do not leak, confidence distributions can.

The 防御: do not expose confidences, or truncate/quantize them before exposure. This is an additional requirement beyond (ε, δ)-DP training.

### Where this fits in Phase 18

Lessons 20-21 are 偏差/公平性. Lesson 22 is 隐私. Lesson 23 is 来源证明 via 水印. Lesson 27 covers the 监管 data-来源证明 layer.

## 使用它

`code/main.py` simulates DP-SGD on a toy binary-classification 数据集. You can sweep the noise multiplier σ and the clipping norm C and track the (ε, δ) budget and the accuracy cost. A "canary 攻击" inserts a unique training example and measures whether a log-loss test can detect it before and after DP.

## 交付它

本课产出 `outputs/skill-dp-audit.md`. 给定 a DP claim on a language 模型 部署, it audits: the (ε, δ) values, the accountant used, the MIA 评估 protocol, and whether confidence-exposure vectors have been assessed.

## 练习

1. 运行 `code/main.py`. Sweep σ in {0.5, 1.0, 2.0} and report the (ε, δ)-accuracy trade-off. Identify the point at which utility collapses.

2. Implement a canary insertion and a log-loss test. Measure detection rate before and after DP-SGD at σ = 1.0.

3. 阅读 Nasr et al. 2025 on training-data extraction. Why does extraction success not collapse under moderate ε? What does this imply about MIA-as-评估?

4. Design a 部署 using PMixED (arXiv:2403.15638) that operates entirely at inference time. What is the threat 模型 that PMixED addresses that DP-SGD does not?

5. Sketch the DP Reversal via LLM Feedback 攻击. Design a countermeasure that limits confidence-score leakage and estimate its 部署 cost.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|-----------------|------------------------|
| DP | "(ε, δ)-差分隐私" | Formal 隐私: 输出 distribution close under neighbouring-数据集 change |
| DP-SGD | "noise-injected SGD" | Gradient clipping + Gaussian noise addition; standard DP training |
| LoRA + DP-SGD | "efficient private 微调" | DP-SGD on low-rank adapters; standard 2025 configuration |
| MIA | "membership inference" | 攻击 that determines whether an example was in training data |
| Canary | "inserted 水印 example" | Unique training example used to measure DP leakage |
| PMixED | "private inference mixture" | Inference-time DP via mixture-of-experts on next-token distributions |
| DP Reversal | "confidence leakage 攻击" | 攻击 that uses a 模型's confidence as an oracle for re-identification |

## 延伸阅读

- [Abadi et al. ， DP-SGD (arXiv:1607.00133)](https://arxiv.org/abs/1607.00133) ， the standard DP training algorithm
- [Carlini et al. ， Extracting Training Data (arXiv:2012.07805)](https://arxiv.org/abs/2012.07805) ， the canonical extraction paper
- [Duan et al. ， Canary MIA on LLMs (arXiv:2402.07841, 2024)](https://arxiv.org/abs/2402.07841) ， limited-success MIA
- [Kowalczyk et al. ， Auditing DP for LLMs (arXiv:2503.06808, March 2025)](https://arxiv.org/abs/2503.06808) ， resolution of the tension
- [PMixED (arXiv:2403.15638)](https://arxiv.org/abs/2403.15638) ， inference-time private prediction
