---
name: skill-fine-tuning-guide-zh
description: Decision tree for when and how to fine-tune LLMs with LoRA and QLoRA
version: 1.0.0
phase: 11
lesson: 8
tags: [fine-tuning, lora, qlora, peft, llm-engineering]
---

# 微调 Decision Guide

Before 微调, try these in order:

```text
1. Prompt engineering (minutes, $0)
2. Few-shot examples in prompt (minutes, $0)
3. RAG for knowledge retrieval (days, $10-100/month)
4. Fine-tuning with LoRA/QLoRA (days, $5-50 per experiment)
5. Full fine-tuning (weeks, $100-10,000 per run)
```

Only move to the next 步骤 if the previous one is measurably insufficient.

## When to fine-tune

- 模型 needs a consistent 输出 风格 or format that prompting cannot achieve
- You're distilling a larger 模型 (GPT-4 质量 from an 8B 模型)
- 延迟 matters and 少样本 examples add too many 词元
- 你need the 模型 to reliably follow a complex 推理 pattern
- 你have 1,000+ high-quality examples of the desired input-output behavior

## When NOT to fine-tune

- 这个模型 already does what you want with the right 提示词
- 你need the 模型 to know facts (use RAG instead)
- 你have fewer than 500 训练 examples (likely to overfit)
- 这个任务 changes frequently (retraining is expensive)
- 你need to audit which 数据 influenced a specific 输出 (微调 is a black box)

## Method selection

|GPU VRAM|7B 模型|13B 模型|70B 模型|
|----------|----------|-----------|-----------|
|16GB (T4)|QLoRA|Not feasible|Not feasible|
|24GB (3090/4090)|QLoRA or LoRA|QLoRA|Not feasible|
|40GB (A100)|LoRA or Full|QLoRA or LoRA|QLoRA|
|80GB (A100/H100)|Full|LoRA or Full|QLoRA or LoRA|

## LoRA configuration checklist

1. Start with r=16, alpha=32 (safe default for most tasks)
2. 目标 q_proj and v_proj first (minimum viable LoRA)
3. 使用学习 速率 2e-4 for QLoRA, 5e-5 for LoRA fp16
4. Set lora_dropout=0.05
5. 训练 for 1-3 epochs (more 风险 过拟合)
6. Evaluate every 100 步骤 on a held-out set
7. Save checkpoints and pick the best by 评估 损失

## Common mistakes

- 训练 for too many epochs (过拟合 after 轮次 2-3 on small datasets)
- Using the same 学习 速率 as full 微调 (LoRA needs higher LR)
- Forgetting to set the pad 词元 (causes NaN losses with Llama 模型)
- Not freezing the base 模型 (defeats the purpose of LoRA)
- Evaluating only on 训练 数据 (always hold out 10-20% for 评估)
- Skipping the 提示词工程 基线 (微调 a problem that prompting already solves)

## 质量 verification

After 训练, compare on 200+ held-out examples:
1. Base 模型 with best 提示词 (基线)
2. Base 模型 with LoRA adapter (your fine-tuned 模型)
3. GPT-4 or Claude with same 提示词 (ceiling)

如果the LoRA 模型 does not beat the prompted 基线, your 训练 数据 or configuration needs work, not more 计算.

## Adapter management

- Keep adapters separate for multi-task serving (swap adapters per request)
- Merge adapters into base 权重 for single-task deployment
- Store adapters on Hugging Face Hub (10-100MB, easy to version and share)
- Test merged 模型 outputs match unmerged outputs before deploying
- 使用TIES-Merging or DARE to combine multiple adapters into one

## 调试 训练

如果损失 does not decrease:
1. Check 学习 速率 (too low for LoRA, try 2e-4)
2. Verify LoRA 层 are actually receiving gradients
3. Confirm base 模型 权重 are frozen
4. Check 数据 formatting (分词器 must match 模型's expected format)

如果损失 decreases but 评估 质量 is bad:
1. 训练 数据 质量 issue (garbage in, garbage out)
2. Overfitting (reduce epochs, increase dropout, add more 数据)
3. Wrong 目标 modules (add MLP 层 for complex tasks)
4. 排序 too low (try r=32 or r=64)
