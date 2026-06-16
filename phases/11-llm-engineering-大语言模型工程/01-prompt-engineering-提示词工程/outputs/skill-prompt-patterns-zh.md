---
name: skill-prompt-patterns-zh
description: Decision framework for choosing the right 提示词 pattern based on 任务 type, reliability requirements, and 目标 模型
version: 1.0.0
phase: 11
lesson: 01
tags: [prompt-engineering, patterns, llm, temperature, cross-model, few-shot, chain-of-thought]
---

# 提示词 Pattern Selection Guide

当building an LLM-powered 特征, choose your 提示词 pattern before writing the 提示词. The pattern determines the structure. The content fills it in.

## Pattern Decision Matrix

|任务 类型|Primary Pattern|Secondary Pattern|Temperature|少样本 Needed?|
|-----------|----------------|-------------------|-------------|-----------------|
|数据 extraction|Template Fill|少样本|0.0|Yes (2-3 examples)|
|分类|少样本|Guardrail|0.0|Yes (3-5 examples)|
|Summarization|Persona + Template|Audience Adapt|0.3|No|
|Code 生成|Persona|Chain-of-Thought|0.0|Optional|
|Creative writing|Persona|Critique|0.7-1.0|No|
|Multi-step 推理|Chain-of-Thought|Decomposition|0.3|Optional|
|问题 回答|Persona + Guardrail|Boundary|0.3|No|
|提示词 生成|Meta-Prompt|Critique|0.7|Yes (1-2 examples)|
|Content moderation|Guardrail + Boundary|少样本|0.0|Yes (5+ examples)|
|Translation/adaptation|Audience Adapt|少样本|0.3|Yes (2-3 examples)|

## When to Use Each Pattern

**Persona Pattern**: use for every 提示词 as a 基线. The only 问题 is how specific to make the role. For generic tasks, a broad role suffices. For domain-specific tasks, the role should name the 领域, seniority level, and 上下文.

**少样本 Pattern**: use when 输出 format matters more than content. If the 模型 needs to produce a specific JSON shape, CSV format, or 分类 标签, examples are more effective than instructions. Rule of thumb: 2-3 examples for simple formats, 5+ for complex or ambiguous formats.

**Chain-of-Thought Pattern**: use for math, logic, multi-step analysis, and any 任务 where the 模型 needs to "show its work." Improves accuracy by 10-40% on 推理 tasks (Wei et al., 2022). Do NOT use for simple factual lookups or extraction -- it wastes 词元.

**Template Fill Pattern**: use for 结构化 extraction where every 输出 must have the same shape. Works best with temperature=0.0 and explicit "N/A" handling for missing fields.

**Critique Pattern**: use when 质量 matters more than speed. The 模型 generates, critiques, and improves. Roughly doubles 词元 成本 but significantly improves accuracy and completeness. Best for high-stakes outputs (reports, recommendations, public-facing content).

**Guardrail Pattern**: use for any user-facing 系统. Always include: scope boundaries, refusal behavior for out-of-scope requests, and explicit "I don't know" handling. Combine with 输入 验证 on the 应用 side.

**Meta-Prompt Pattern**: use to 生成 prompts for new tasks. Instead of writing a 提示词 from scratch, describe the 任务 and let the 模型 write the 提示词. Then test and iterate. Saves time on initial 提示词 development.

**Decomposition Pattern**: use for complex problems that benefit from divide-and-conquer. The 模型 breaks the problem into parts, solves each, and combines. Most effective for tasks with 3-7 sub-problems.

**Audience Adaptation Pattern**: use when the same content needs to serve different audiences. Specify the audience explicitly -- do not rely on the 模型 guessing from 上下文.

**Boundary Pattern**: use for 生产 systems that must NEVER 答案 certain types of 问题. Stronger than 护栏 because it defines a hard scope with an exact refusal 消息. Essential for compliance-sensitive domains.

## Cross-Model Compatibility

Patterns ranked by how consistently they work across GPT-4o, Claude 3.5 Sonnet, Gemini 1.5 Pro, and Llama 3:

|Pattern|Cross-Model Consistency|Notes|
|---------|------------------------|-------|
|少样本|Very high|Examples transfer well across all 模型|
|Template Fill|Very high|Explicit structure leaves little room for divergence|
|Chain-of-Thought|High|All major 模型 support "think 步骤 by 步骤"|
|Persona|High|Works everywhere but different 模型 respond to different role specificity levels|
|Guardrail|Moderate|Claude follows 护栏 most strictly; GPT-4o sometimes drifts in long conversations|
|Critique|Moderate|质量 of self-critique varies significantly by 模型|
|Meta-Prompt|Moderate|GPT-4o and Claude produce different 提示词 styles|
|Boundary|Low-Moderate|Refusal behavior varies; test per 模型|

## Common Mistakes

1. **Using Chain-of-Thought for everything**: CoT adds 词元 and 延迟. Only use it when 推理 步骤 are needed.
2. **Too many constraints**: more than 5-7 constraints and the 模型 starts dropping some. Prioritize the 3 most important.
3. **Contradictory persona + constraints**: "You are a creative writer" + "Never use metaphors" confuses the 模型.
4. **No temperature specification**: leaving temperature at default (usually 1.0) when you need deterministic 输出.
5. **Copy-pasting prompts across 模型**: always test. A 提示词 tuned for GPT-4o may underperform on Claude and vice versa.
6. **Ignoring 系统 消息**: putting everything in the 用户 消息 instead of using the 系统 消息 for persistent rules.
7. **Over-relying on negative constraints**: "Do NOT do X, Y, Z, A, B, C" is less effective than "ONLY do W." Positive framing gives the 模型 a clear 目标.

## Reliability 目标

|Use Case|Pattern Combination|Expected Accuracy|词元 成本|
|----------|-------------------|-------------------|------------|
|生产 extraction|Template + 少样本|95%+|Low (500-1K)|
|User-facing Q&A|Persona + Guardrail + Boundary|90%+|Medium (1-2K)|
|Code 生成|Persona + Chain-of-Thought|85%+|Medium (1-3K)|
|Content 生成|Persona + Critique|90%+ 质量|High (2-4K, double pass)|
|分类|少样本 + Guardrail|95%+|Low (300-800)|
|Complex analysis|Decomposition + Chain-of-Thought|85%+|High (3-5K)|
