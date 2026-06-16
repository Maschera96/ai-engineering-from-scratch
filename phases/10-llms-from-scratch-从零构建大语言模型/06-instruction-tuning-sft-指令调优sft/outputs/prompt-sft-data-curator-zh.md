---
name: prompt-sft-data-curator-zh
description: Design and curate instruction datasets for supervised 微调
version: 1.0.0
phase: 10
lesson: 6
tags: [sft, instruction-tuning, fine-tuning, data-curation, alignment]
---

# SFT 数据 Curator

当designing an instruction-tuning 数据集 for a specific capability (code 生成, math, conversation, 安全), use this framework to plan 数据 collection, define 质量 criteria, and structure the 训练 流水线.

## 输入 Requirements

Provide:
- **目标 capability** (e.g., "Python code 生成", "medical Q&A", "multi-turn conversation")
- **Base 模型** (e.g., Llama 3 8B, Mistral 7B, Qwen 2.5 72B)
- **预算** (annotation 小时, API 成本 for synthetic 生成)
- **Format preference** (Alpaca, ShareGPT, ChatML)

## 步骤 1: 数据集 Design

### Size Guidelines

|质量 Level|Examples Needed|Expected Outcome|
|--------------|----------------|------------------|
|Research prototype|1,000-5,000|LIMA-quality: comparable to larger datasets if examples are expert-written|
|生产 v1|10,000-50,000|Stanford Alpaca level: solid instruction following across common tasks|
|生产 v2|50,000-200,000|Vicuna/Llama 2 Chat level: robust multi-turn, 领域 coverage|

质量 always beats quantity. 1,000 expert-written examples (LIMA, May 2023) matched 模型 训练后的 on 50,000+ examples. Prioritize:

1. **Diversity** -- cover the full range of 目标 capabilities
2. **Accuracy** -- every 响应 must be factually correct
3. **Clarity** -- 响应 should be concise and well-structured
4. **Difficulty 梯度** -- include easy, medium, and hard examples

### Diversity Checklist

For a general-purpose 助手:
- Open-ended 问题 (20%)
- Factual Q&A (20%)
- Creative writing (10%)
- Code 生成 (15%)
- 推理 and math (15%)
- Summarization (10%)
- Instruction following with constraints (10%)

Adjust percentages for domain-specific 模型. A coding 助手 might allocate 60% to code 生成 and 20% to code explanation.

## 步骤 2: 数据 Format

### Alpaca Format (single-turn)

```json
{
  "instruction": "Write a function that reverses a string in Python.",
  "input": "",
  "output": "def reverse_string(s):\n    return s[::-1]"
}
```

使用when: single-turn tasks, simple instruction-response pairs, rapid prototyping.

### ShareGPT Format (multi-turn)

```json
{
  "conversations": [
    {"from": "system", "value": "You are a Python expert."},
    {"from": "human", "value": "How do I reverse a string?"},
    {"from": "gpt", "value": "Use slicing: s[::-1]"},
    {"from": "human", "value": "What about for a list?"},
    {"from": "gpt", "value": "Same syntax works: my_list[::-1]"}
  ]
}
```

使用when: conversational applications, multi-turn 上下文 is important.

### ChatML Format (with special 词元)

```text
<|im_start|>system
You are a Python expert.<|im_end|>
<|im_start|>user
How do I reverse a string?<|im_end|>
<|im_start|>assistant
Use slicing: s[::-1]<|im_end|>
```

使用when: targeting 模型 that use ChatML natively (Qwen, Yi).

## 步骤 3: 质量 Criteria

### Per-Example Checks

1. **响应 relevance**: Does the 响应 actually 答案 the instruction?
2. **Factual accuracy**: Are all claims verifiable and correct?
3. **Completeness**: Does the 响应 fully address the instruction?
4. **Conciseness**: Could the same information be conveyed in fewer words?
5. **Format consistency**: Does the 响应 follow the expected 风格?

### Red Flags (reject the example)

- 响应 contradicts itself
- 响应 includes harmful content without refusal
- 响应 hallucinates facts or citations
- Instruction is ambiguous and 响应 doesn't clarify
- 响应 is a copy of the instruction rephrased

### Dataset-Level Checks

- No more than 5% of examples from any single 来源/template
- At least 80% of 响应 词元 are meaningful (not filler)
- Average 响应 length is 50-200 词元 (avoid very short or very long)
- 系统 提示词 diversity: at least 10 different 系统 prompts represented

## 步骤 4: 训练 Configuration

|参数|Recommended Range|Notes|
|-----------|------------------|-------|
|学习 速率|1e-5 to 5e-5|Lower for larger 模型 (1e-5 for 70B, 5e-5 for 7B)|
|Epochs|1-3|Monitor 验证 损失, stop at first sign of increase|
|批次 size|32-128|规模 with 梯度 accumulation if GPU-limited|
|预热|0-5% of 步骤|Less critical than 预训练|
|权重 衰减|0.0-0.1|Optional for short 微调 runs|
|损失 masking|响应 词元 only|掩码 instruction and 系统 提示词 词元|
|Pre-training 数据 mixing|2-5%|Mix raw 文本 to prevent catastrophic forgetting|

## 步骤 5: 评估 协议

After 训练, evaluate on:

1. **Instruction following 速率**: Percentage of test prompts where the 模型 produces a relevant, complete 响应
2. **Forgetting 分数**: Perplexity on a held-out general 文本 语料库 compared to the base 模型
3. **Format compliance**: Percentage of 响应 that follow the expected chat format
4. **MT-Bench or AlpacaEval**: Standard benchmarks for instruction-tuned 模型
5. **Domain-specific 评估**: Custom 评估 for your 目标 capability

### Warning Signs

- 验证 损失 increases after 轮次 1: you're 过拟合, reduce epochs or increase 数据
- Forgetting 分数 increases > 15%: 学习 速率 too high or too many epochs
- 模型 reproduces 训练 examples verbatim: severe 过拟合, needs more diverse 数据
- 模型 refuses benign instructions: over-trained on 安全 数据, rebalance the 数据集
