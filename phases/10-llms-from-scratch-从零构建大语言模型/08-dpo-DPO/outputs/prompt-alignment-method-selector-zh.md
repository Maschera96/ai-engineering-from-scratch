---
name: prompt-alignment-method-selector-zh
description: Choose the right 对齐 method (SFT, RLHF, DPO, KTO, ORPO, SimPO) for your use case
version: 1.0.0
phase: 10
lesson: 8
tags: [alignment, dpo, rlhf, kto, orpo, simpo, preference-optimization, fine-tuning]
---

# 对齐 Method Selector

当choosing an 对齐 method for a 语言模型, use this framework to evaluate your 数据, 计算, and 质量 requirements, then select the method that best fits your constraints.

## 输入 Requirements

Provide:
- **Base 模型** (e.g., Llama 3 8B, Mistral 7B, Qwen 2.5 72B)
- **Starting point** (base 模型, or already SFT'd?)
- **Available 数据** (instruction pairs, preference pairs, 非成对 ratings, or none)
- **计算 预算** (GPU 小时, number of GPUs)
- **质量 目标** (good enough for prototype, competitive with open-source, state-of-the-art)
- **时间line** (days, weeks, months)

## Decision Matrix

### Quick Selection

|Your Situation|Recommended Method|Why|
|---------------|-------------------|-----|
|No preference 数据, only instruction pairs|SFT only|You can't align without preference 信号|
|< 5,000 preference pairs, limited 计算|DPO|Simpler 流水线, works well with small 数据|
|非成对 feedback (thumbs up/down only)|KTO|Only method that works without pairwise comparisons|
|Want 对齐 in a single 训练 run|ORPO|Combines SFT + 对齐, no 参考 模型|
|Memory-constrained (can't fit 参考 模型)|SimPO|No 参考 模型 needed|
|Large-scale, multi-objective 对齐|RLHF (PPO)|Separate 奖励模型 captures complex preferences|
|Iterative 对齐 with online 数据|RLHF (PPO)|Can 生成, 速率, and retrain in a 循环|
|Post-RLHF refinement|DPO|Fine-tune an RLHF 模型 on targeted preferences|

### Detailed Comparison

|Method|数据 Requirement|模型 in 内存|训练 Loops|Stability|Best 规模|
|--------|-----------------|-----------------|----------------|-----------|------------|
|SFT|Instruction pairs (10K+)|1|1|High|Any|
|RLHF|Preference pairs (20K+)|3-4|3|Low|Large (70B+)|
|DPO|Preference pairs (5K+)|2|2 (SFT + DPO)|High|Small-Medium (7B-70B)|
|KTO|非成对 ratings (5K+)|2|2 (SFT + KTO)|High|Any|
|ORPO|Preference pairs (10K+)|1|1|High|Small-Medium|
|SimPO|Preference pairs (5K+)|1|2 (SFT + SimPO)|High|Small-Medium|

## Method-Specific Configuration

### SFT

- **When to stop**: After 1-3 epochs or when 验证 损失 stops decreasing
- **Key hyperparameter**: 学习 速率 (1e-5 to 5e-5, lower for bigger 模型)
- **Critical detail**: 掩码 instruction 词元 in the 损失
- **Gotcha**: More than 3 epochs causes memorization; mix in 2-5% 预训练 数据

### RLHF (PPO)

- **When to use**: You have 20K+ comparison pairs, need multi-objective 对齐, or want iterative online 学习
- **Key hyperparameters**: KL coefficient (0.01-0.05), PPO clip 比例 (0.1-0.3), 学习 速率 (5e-6 to 3e-5)
- **Critical detail**: 奖励模型 should be >= 策略 模型 size
- **Gotcha**: PPO is unstable; monitor KL divergence and 奖励 曲线 continuously

### DPO

- **When to use**: You have preference pairs and want a simpler 流水线 than RLHF
- **Key hyperparameter**: Beta (0.1-0.5; lower = more deviation from 参考 allowed)
- **Critical detail**: 参考 模型 must be a frozen copy of the SFT checkpoint
- **Gotcha**: Very sensitive to beta; run a sweep over [0.05, 0.1, 0.2, 0.5]

### KTO

- **When to use**: You only have "good" or "bad" 标签s without pairwise comparisons
- **Key hyperparameter**: Beta (same as DPO), 损失 aversion multiplier (1.5x on bad 响应)
- **Critical detail**: Needs roughly balanced good/bad examples (40-60% split)
- **Gotcha**: Without pairs, the 梯度 信号 is weaker; may need more 数据 than DPO

### ORPO

- **When to use**: You want to skip SFT entirely and go straight from base to aligned
- **Key hyperparameter**: Lambda (权重 of the preference term vs SFT term)
- **Critical detail**: Needs both instruction 标签s AND preference pairs in one 数据集
- **Gotcha**: Combined 目标 can be hard to balance; if SFT 损失 dominates, 对齐 is weak

### SimPO

- **When to use**: Memory-constrained setup where you can't hold a 参考 模型
- **Key hyperparameter**: Beta, gamma (length 归一化 exponent)
- **Critical detail**: Length 归一化 prevents the 模型 from favoring short 响应
- **Gotcha**: Without a 参考 模型 anchor, the 模型 can drift further; monitor carefully

## 流水线 Templates

### Template 1: Fast Prototype (1-2 days)

```text
Base Model -> SFT (1 epoch, 10K examples) -> DPO (3 epochs, 5K pairs)
```

计算: ~4 GPU-hours for 7B 模型 on A100
质量: Solid instruction following, basic preference 对齐

### Template 2: 生产 质量 (1-2 weeks)

```text
Base Model -> SFT (2 epochs, 50K examples) -> DPO (5 epochs, 20K pairs) -> Eval -> Iterate
```

计算: ~40 GPU-hours for 7B, ~200 GPU-hours for 70B
质量: Competitive with open-source RLHF 模型

### Template 3: State-of-the-Art (1-3 months)

```text
Base Model -> SFT (2 epochs, 100K+ examples) -> RLHF (PPO, 50K+ pairs) -> DPO (targeted refinement) -> Eval -> Iterate
```

计算: ~500+ GPU-hours for 70B
质量: 方法ing frontier 模型 对齐

### Template 4: Minimal 数据 (1-2 days)

```text
Base Model -> SFT (1 epoch, 5K examples) -> KTO (unpaired thumbs up/down from users)
```

计算: ~2 GPU-hours for 7B
质量: Better than SFT-only with minimal 数据 collection overhead

## 评估 协议

After 对齐, evaluate across these 维度:

1. **Preference win 速率**: Compare aligned 模型 vs SFT 模型 on 200+ test prompts with human judges. 目标: > 60% win 速率.
2. **基准 retention**: MMLU, HumanEval, or domain-specific benchmarks. Should not drop > 5% from SFT 基线.
3. **MT-Bench or AlpacaEval**: Standard 对齐 质量 benchmarks. Compare against published baselines.
4. **安全 评估**: Test against adversarial prompts, jailbreaks, and harmful request categories.
5. **响应 diversity**: Measure 熵 of 响应 across 100 prompts. Low 熵 = mode 崩塌.

## Common Failure Modes

|Symptom|Cause|Method-Specific 修复|
|---------|-------|-------------------|
|Verbose, padded 响应|奖励模型 / implicit 奖励 favors length|DPO: increase beta. RLHF: add length penalty. SimPO: adjust gamma.|
|模型 agrees with everything|Sycophancy from preference 数据 偏差|Add preference pairs where the correct 响应 disagrees with the 用户|
|Refuses benign requests|Over-对齐 on 安全 数据|Reduce 安全 example proportion, add more benign-refusal pairs|
|Outputs are nearly identical to SFT|Beta too high (DPO/KTO) or KL coefficient too high (PPO)|Lower beta / KL coefficient; the 模型 isn't 学习|
|训练 损失 oscillates|学习 速率 too high or insufficient 数据|Reduce lr by 2-3x; increase preference 数据|
