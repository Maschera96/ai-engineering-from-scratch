---
name: skill-llm-evaluation-zh
description: Decision framework for choosing the right LLM 评估 strategy based on 任务 type, 预算, and requirements
version: 1.0.0
phase: 10
lesson: 10
tags: [evaluation, evals, benchmarks, llm-as-judge, elo, metrics]
---

# LLM 评估 Strategy

当evaluating an LLM 系统, apply this decision framework to choose the right approach.

## When to use each 评估 type

**Benchmarks (MMLU, HumanEval, SWE-bench):** You are doing initial 模型 selection. You need to narrow 10 candidate 模型 to 3. Benchmarks give rough 排序 at zero 成本. Do not use benchmarks as your final 评估.

**Custom evals:** You are building for 生产. You have a specific 任务 with specific failure modes. Custom evals are the only 评估 that predicts real-world performance. Minimum 50 test cases for prototype, 200+ for 生产.

**LLM-as-judge:** Your 任务 is open-ended (summarization, writing, conversation). Exact match and 词元 overlap 指标 are too rigid. LLM-as-judge 成本 ~$0.01 per judgment and agrees with humans ~80% of the time. Always use a rubric, not a vague 提示词.

**Human evals:** The stakes are high and automated 指标 disagree. Human 评估 is the ground truth but 成本 $0.10-$2.00 per judgment. Reserve for ambiguous cases and periodic calibration of automated 指标.

**ELO from pairwise comparisons:** You are comparing multiple 模型 on the same 任务. Pairwise is more reliable than absolute scoring because humans (and LLM judges) are better at relative judgments.

## Scoring 函数 selection

- **Exact match**: 分类, entity extraction, 结构化输出 with known answers
- **词元 F1**: extraction tasks where partial credit matters
- **ROUGE-L**: summarization, translation
- **BLEU**: machine translation
- **LLM-as-judge**: open-ended 生成, conversational 质量, helpfulness
- **Execution-based**: code 生成 (run the code, check if tests pass)
- **模式 compliance**: 结构化输出 (does the JSON match the 模式?)

## Red flags in 评估 design

- 评估 set smaller than 50 cases: results are statistically meaningless
- No 边 cases: you are measuring happy-path performance, which is always higher than real-world
- Single 指标: different 指标 tell different stories, use at least two
- No versioning: you cannot track improvement without versioned 评估 sets
- 评估 set contamination: never include 评估 examples in 微调 数据 or 少样本 prompts
- 测试 only one 模型: you need a 基线 (even a simple heuristic) for comparison

## 评估 流水线 checklist

1. Define the 任务 precisely (not "答案 问题" but "classify support tickets into 5 categories")
2. Create test cases across happy path, 边 cases, and known regressions
3. Select 2-3 scoring 函数 appropriate for the 任务 type
4. Set pass/fail thresholds based on 生产 requirements
5. Automate execution: one command runs the full suite
6. Version everything: test cases, scoring 函数, prompts, 模型 versions
7. 运行on every change: 提示词 updates, 模型 swaps, code deployments
8. Track trends: a single 分数 is 噪声, a trendline is 信号
9. Calibrate against human judgment quarterly
10. Add 回归 cases whenever a 生产 failure is discovered
