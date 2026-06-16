---
name: skill-cot-patterns-zh
description: Decision framework for choosing the right 推理 technique based on 任务 complexity, accuracy requirements, and 成本 constraints
version: 1.0.0
phase: 11
lesson: 02
tags: [chain-of-thought, few-shot, self-consistency, tree-of-thought, react, reasoning, prompting]
---

# 推理 Technique Selection Guide

当you need an LLM to 原因 through a problem, choose the technique before writing the 提示词. The technique determines the 推理 架构. The 提示词 fills it in.

## Quick Decision Tree

1. Is the 任务 a simple factual lookup or single-step 分类?
   - Yes: use **零样本**. CoT adds 成本 with no accuracy gain.
   - No: continue.

2. Does the 任务 require multi-step 推理 (math, logic, planning)?
   - Yes: use **Chain-of-Thought**. Continue to 步骤 3.
   - No: use **少样本** if format matters, 零样本 if it does not.

3. Is a single 推理 错误 acceptable?
   - Yes: use **少样本 CoT** (single 样本, temperature 0.0).
   - No: use **self-consistency** (N=5, temperature 0.7). Continue to 步骤 4.

4. Is the problem a search/planning problem with many possible paths?
   - Yes: use **Tree-of-Thought**.
   - No: self-consistency is sufficient.

5. Does the 任务 require external information or computation?
   - Yes: use **ReAct** (推理 + 工具 calls).
   - No: pure 推理 techniques are sufficient.

## Technique Matrix

|Technique|Accuracy Lift|成本 Multiplier|延迟|Best For|
|-----------|--------------|-----------------|---------|----------|
|零样本|基线|1x|~1s|Simple tasks, factual Q&A|
|少样本|+5-15%|1.2x|~1s|Format 匹配, 分类|
|零样本 CoT|+10-20%|1.3x|~1.5s|Quick 推理 boost|
|少样本 CoT|+15-25%|1.5x|~2s|Math, logic, multi-step|
|Self-Consistency (N=5)|+2-5% over CoT|5x|~5s|High-stakes 推理|
|Self-Consistency (N=10)|+1-2% over N=5|10x|~10s|Critical decisions only|
|Tree-of-Thought|Task-dependent|10-40x|~30s+|Search, planning, puzzles|
|ReAct|Task-dependent|3-10x|~5-15s|Knowledge-grounded tasks|
|提示词 Chaining|+5-10% over single|2-5x|~5-10s|Complex multi-part tasks|

## Model-Specific Guidance

### GPT-4o / GPT-4.1
- Strong 基线 推理. 零样本 CoT often sufficient.
- 少样本 CoT with 3 examples hits 95% on GSM8K.
- Self-consistency gives marginal gains (95% to 97%) -- only worth it for critical tasks.
- Supports 结构化输出 natively for 答案 extraction.

### Claude 3.5 Sonnet / Claude 3.7 Sonnet
- Excellent at following 结构化 提示词 formats (XML tags).
- 少样本 CoT with XML-delimited examples works best.
- Extended thinking (Claude 3.7) is native CoT -- no need to 提示词 for it.
- Self-consistency is effective because Claude's 推理 varies well at temperature 0.7.

### Llama 3.1/3.3 70B
- Benefits most from 少样本 CoT (larger accuracy gap vs 零样本).
- Self-consistency with N=5 recommended for 推理 tasks.
- Needs more explicit format instructions than commercial 模型.
- ToT is expensive on local 推理 -- consider only for 批次 processing.

### Gemini 2.5 Pro
- Strong at multi-step 推理 out of the box.
- Thinking mode provides built-in CoT without 提示词工程.
- 少样本 examples help with format consistency more than accuracy.
- Large 上下文 window (1M) makes example-heavy 少样本 practical.

## Anti-Patterns

**CoT for simple tasks**: asking "What is 2+2? Let's think 步骤 by 步骤" wastes 词元. The 模型 gets simple arithmetic right without 推理 traces. CoT helps when there are 3+ 步骤.

**Self-consistency at temperature 0.0**: all N 样本 will be identical. You must use temperature > 0 (0.5-0.8 recommended) for diverse 推理 paths.

**ToT for everything**: ToT requires O(b^d) LLM calls where b=branching factor and d=深度. A tree with b=3, d=3 needs up to 39 calls. Reserve for problems where cheaper techniques fail.

**少样本 with bad examples**: examples with 推理 错误 teach the 模型 to make those 错误. Every example must be verified. One wrong example can reduce accuracy more than zero examples.

**Extracting answers without a consistent format**: self-consistency requires comparing answers across 样本. If the 答案 format varies ("$18", "18 dollars", "eighteen"), voting fails. Always enforce: "The 答案 is [number]."

## 成本 优化

For a 生产 系统 handling 10,000 查询/day at GPT-4o pricing ($2.50/1M 输入, $10/1M 输出):

|Technique|Avg 词元/查询|Daily 成本|Accuracy|
|-----------|-----------------|------------|----------|
|零样本|~200|~$5|78%|
|少样本 CoT|~600|~$15|95%|
|Self-Consistency (N=5)|~3,000|~$75|97%|
|ToT (b=3, d=2)|~6,000|~$150|Task-dependent|

这个cost-optimal strategy for most applications: start with 少样本 CoT. Add self-consistency only for 查询 where confidence is low (the escalation pattern from the 动手构建 section).

## Integration with 提示词 Chaining

推理 techniques compose with 提示词 chaining:

**链 步骤 1** (Extract): 零样本, temperature 0.0
**链 步骤 2** (原因): 少样本 CoT, temperature 0.0
**链 步骤 3** (Verify): self-consistency with N=3, temperature 0.7

这three-step 链 成本 ~3x a single CoT call but catches extraction 错误, 推理 错误, and provides a confidence 分数 from the verification 步骤.

## When to Move Beyond Prompting

如果you are spending more time engineering prompts than writing 应用 code, consider:

1. **Fine-tuning**: if you have 500+ 标签ed examples and the 任务 is narrow
2. **DSPy compilation**: if you want automated 提示词 优化
3. **智能体 frameworks**: if the 任务 requires multi-turn 工具使用 (Phase 14)
4. **RAG**: if the 模型 needs access to private/current knowledge (Lessons 06-07)

Prompting techniques are the foundation. They work with any 模型, any provider, and require no 训练 数据. But they have limits. Knowing when to graduate to the next level is as important as mastering the techniques themselves.
