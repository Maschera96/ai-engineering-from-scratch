---
name: prompt-reward-model-designer-zh
description: Design 奖励模型 训练 pipelines for RLHF 对齐
version: 1.0.0
phase: 10
lesson: 7
tags: [rlhf, reward-model, ppo, alignment, human-feedback, preference-learning]
---

# 奖励 模型 Designer

当building an RLHF 流水线 to align a 语言模型 toward a 目标 behavior (helpfulness, coding ability, 安全, honesty), use this framework to design the 数据 collection 协议, 训练 the 奖励模型, and configure PPO.

## 输入 Requirements

Provide:
- **目标 behavior** (e.g., "helpful and harmless 助手", "expert Python coder", "medical Q&A with 安全")
- **Base 模型** (e.g., Llama 3 8B after SFT, Mistral 7B Chat)
- **奖励模型 size** (typically same size or larger than the 策略 模型)
- **Annotation 预算** (human 小时 or comparison pairs available)
- **计算 预算** (GPU 小时 for 奖励模型 训练 + PPO)

## 步骤 1: Preference 数据 Collection

### Annotation 协议

1. **提示词 selection**: 样本 from the SFT 训练 分布 plus out-of-distribution prompts (10-20% novel)
2. **响应 生成**: 生成 2-4 响应 per 提示词 using the SFT 模型 with different temperatures (0.3, 0.7, 1.0)
3. **Comparison format**: Show annotators exactly 2 响应 and ask "Which 响应 is better?"
4. **Criteria rubric**: Define what "better" means for your use case

### Rubric Template

|Criterion|权重|描述|
|-----------|--------|-------------|
|Helpfulness|40%|Does it 答案 the 问题 completely and correctly?|
|Harmlessness|25%|Does it avoid harmful, biased, or misleading content?|
|Honesty|20%|Does it acknowledge uncertainty rather than hallucinate?|
|Conciseness|15%|Is the 响应 an appropriate length for the 问题?|

Adjust 权重 for your use case. A coding 助手 might 权重 correctness at 60% and conciseness at 20%.

### 数据 Size Guidelines

|规模|Comparison Pairs|Annotator 小时|Expected RM Accuracy|
|-------|-----------------|-----------------|---------------------|
|Minimum viable|5,000-10,000|400-800|60-65%|
|生产 v1|20,000-50,000|1,600-4,000|65-72%|
|生产 v2|100,000-500,000|8,000-40,000|72-78%|

InstructGPT used 33,000 comparisons from 40 contractors. Anthropic's initial paper used 22,000 from 20 annotators. Inter-annotator agreement is typically 70-75% -- the 奖励模型 cannot exceed human agreement levels.

### 质量 Control

- **Agreement filtering**: Discard pairs where fewer than 70% of annotators agree
- **Annotator calibration**: Run calibration rounds with known-good pairs before 真实 annotation
- **偏差 detection**: Monitor if annotators consistently prefer longer 响应, formal language, or specific patterns
- **Adversarial examples**: Include 5-10% examples designed to catch annotators who are not reading carefully

## 步骤 2: 奖励 模型 架构

### 架构 Decisions

|Decision|Recommendation|Rationale|
|----------|---------------|-----------|
|Base 架构|Same transformer as the 策略|权重 initialization from SFT checkpoint gives strong starting 特征s|
|输出 头|Single linear projection from last 隐藏 状态|Scalar 奖励 from the most complete position representation|
|模型 size|>= 策略 模型 size|Smaller RM produces unreliable signals that destabilize PPO|
|Initialization|SFT checkpoint with new 输出 头|Pre-trained 特征s capture language 质量 already|

### 训练 Configuration

|参数|Range|Notes|
|-----------|-------|-------|
|学习 速率|1e-5 to 5e-5|Lower than SFT because the 任务 is simpler|
|Epochs|1-3|Overfitting is a major 风险 with limited comparison 数据|
|批次 size|64-256|Each "example" is a pair, so effective 数据 is 2x|
|损失函数|Bradley-Terry: -log(sigmoid(r_preferred - r_rejected))|Standard for pairwise comparisons|
|验证 split|10-20%|Monitor accuracy on held-out pairs|

### 评估 指标

1. **Pairwise accuracy**: What fraction of held-out preference pairs does the RM 排序 correctly? 目标: > 65%
2. **Margin 分布**: Plot the 分布 of (r_preferred - r_rejected). Should be centered above 0 with few negatives.
3. **Calibration**: Is sigmoid(r_preferred - r_rejected) close to the actual human preference 概率?
4. **OOD 泛化**: Test on prompts from a different 分布 than 训练. Accuracy should drop < 10%.

## 步骤 3: PPO Configuration

### Hyperparameters

|参数|Typical Value|Effect of Being Too High|Effect of Being Too Low|
|-----------|--------------|-------------------------|------------------------|
|KL coefficient (beta)|0.01-0.05|模型 barely learns, stays too close to SFT|奖励 hacking, degenerate outputs|
|学习 速率|5e-6 to 3e-5|训练 instability, divergence|Slow convergence, wasted 计算|
|Clip 比例 (epsilon)|0.1-0.3|Large, potentially destabilizing updates|Very conservative updates, slow 学习|
|PPO epochs per 批次|1-4|Overfitting to current 批次|Underutilizing each 批次|
|生成 批次 size|128-512|内存 issues|Noisy 梯度 estimates|
|Max 响应 length|256-1024|Slow 生成, 内存 issues|Truncates useful 响应|

### Monitoring Dashboard

Track these 指标 during PPO 训练:

1. **Mean 奖励**: Should increase over 训练. Plateau is fine; decrease means instability.
2. **KL divergence**: Should stay below 10-20 nats. Spike = 奖励 hacking.
3. **响应 length**: Should stay stable. Monotonic increase = verbosity 奖励 hacking.
4. **熵**: 词元 分布 熵 should decrease slowly. Rapid decrease = mode 崩塌.
5. **奖励模型 agreement**: 分数 PPO 响应 with the 奖励模型; agreement should improve.

### Red Flags During PPO

|Symptom|Likely Cause|修复|
|---------|-------------|-----|
|奖励 increases but outputs degrade|奖励 hacking|Increase KL coefficient, retrain RM on adversarial examples|
|KL divergence explodes|学习 速率 too high or KL coefficient too low|Reduce lr, increase beta|
|响应 length grows monotonically|RM rewards verbosity|Add length penalty to 奖励, retrain RM with length-controlled pairs|
|All 响应 become identical|Mode 崩塌|Increase 生成 temperature, reduce PPO epochs|
|奖励 oscillates wildly|PPO instability|Reduce 学习 速率, increase clip 比例|

## 步骤 4: End-to-End 验证

Before deploying an RLHF-trained 模型:

1. **A/B test vs SFT**: Run the SFT and RLHF 模型 on 200+ test prompts. Have 3+ evaluators compare 响应. The RLHF 模型 should win > 60% of the time.
2. **安全 评估**: Test on known adversarial prompts (jailbreaks, harmful requests). The RLHF 模型 should refuse appropriately.
3. **回归 check**: Run standard benchmarks (MMLU, HumanEval, MT-Bench) to confirm the RLHF 模型 hasn't lost core capabilities.
4. **Forgetting check**: Measure perplexity on a general 文本 语料库. Increase should be < 10% vs the SFT 模型.
5. **Length analysis**: Compare average 响应 length between SFT and RLHF 模型. If RLHF is > 50% longer, the 奖励模型 likely has a verbosity 偏差.
