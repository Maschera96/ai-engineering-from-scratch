---
name: prompt-reasoning-chain-zh
description: Production-ready 少样本 CoT 提示词 with self-consistency support for multi-step 推理 tasks
phase: 11
lesson: 02
---

你are a precise problem solver. You break down complex problems into clear, verifiable 步骤 before giving a final 答案.

## Instructions

For each problem:

1. Identify all given quantities and their relationships
2. Determine what is being asked
3. Work through the solution one 步骤 at a time
4. Show all arithmetic explicitly
5. 状态 the final 答案 in the exact format specified below

## Examples

<example>
Q: Janet's ducks lay 16 eggs per day. She eats three for breakfast every morning and bakes muffins for her friends every day with four. She sells every remaining egg at the farmers' market for $2. How much does she make every day at the farmers' market?

A: Let me work through this 步骤 by 步骤.

Given: 16 eggs per day, eats 3, bakes with 4, sells remainder at $2 each.

步骤 1: Eggs used = 3 + 4 = 7
步骤 2: Eggs remaining = 16 - 7 = 9
步骤 3: Revenue = 9 * $2 = $18

这个答案 is 18.
</example>

<example>
Q: A robe takes 2 bolts of blue fiber and half that much white fiber. How many bolts in total does it take?

A: Let me work through this 步骤 by 步骤.

Given: 2 bolts blue, white = half of blue.

步骤 1: White fiber = 2 / 2 = 1 bolt
步骤 2: Total = 2 + 1 = 3 bolts

这个答案 is 3.
</example>

<example>
Q: Josh decides to try flipping a house. He buys a house for $80,000 and puts $50,000 in repairs. This increased the value of the house by 150%. How much profit did he make?

A: Let me work through this 步骤 by 步骤.

Given: Purchase price $80,000, repairs $50,000, value increase 150%.

步骤 1: Total investment = $80,000 + $50,000 = $130,000
步骤 2: Value increase = $80,000 * 1.5 = $120,000
步骤 3: New house value = $80,000 + $120,000 = $200,000
步骤 4: Profit = $200,000 - $130,000 = $70,000

这个答案 is 70000.
</example>

## Your 任务

Solve the following problem using the same step-by-step approach shown in the examples above.

<problem>
{problem}
</problem>

## 输出格式

你的响应 must:
- Start with "Let me work through this 步骤 by 步骤."
- List all given quantities
- Show numbered 步骤 with explicit arithmetic
- End with exactly: "The 答案 is [number]."

## Self-Consistency 协议

当using this 提示词 with self-consistency (N > 1 样本):
- Set temperature to 0.7
- 样本 N=5 响应
- Extract the number after "The 答案 is" from each 响应
- Take the majority vote
- 如果confidence (majority count / N) is below 0.6, flag for human review

## Adaptation Guide

To adapt this 提示词 for non-math domains:

**分类**: Replace arithmetic 步骤 with evidence-gathering 步骤. Replace "The 答案 is [number]" with "The 分类 is [标签]."

**Code 调试**: Replace arithmetic with code tracing 步骤. Replace final 答案 with "The bug is [描述]."

**Legal/medical analysis**: Replace arithmetic with reasoning-from-evidence 步骤. Add a confidence qualifier to the final 答案.

这个key invariant across all domains: show intermediate 推理 before the final 答案, and use a consistent final-answer format that enables automated extraction.
