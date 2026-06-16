---
name: lm-eval-harness-zh
description: 最小 language model evaluation harness，包含 JSONL task spec、五种 metrics、可替换 adapter，以及 leaderboard JSON output。
version: 1.0.0
phase: 19
lesson: 49
tags: [evaluation, metrics, leaderboard, harness]
---

## 何时使用

用固定任务集比较两个 models、两个 checkpoints，或两个 prompt templates。凡是要发布并且要长期监控的东西，都适用。

## Task spec

每个 example 一行 JSONL：

```json
{"id": "ex-001", "prompt": "...", "targets": ["..."], "metric": "exact_match", "extras": {}}
```

同一个文件中的所有 examples 共用一种 metric。文件名就是 task name。

## Metrics

| Metric | Signature | Use for |
|--------|-----------|---------|
| exact_match | normalize lower + whitespace, equality | 算术、factoid answers |
| substring_contains | target must appear in normalized prediction | 带 anchor words 的自由形式生成 |
| multiple_choice | first letter match | A/B/C/D style questions |
| rouge_l | LCS F1 over tokenized text | 摘要、paraphrase |
| code_exec | run prediction's `f` on io_pairs, count matches | 代码生成 |

所有 metrics 都返回 [0.0, 1.0] 范围内的 float。Task score 是均值。

## Adapter

```python
class Adapter(Protocol):
    name: str
    def generate(self, prompts: list[str]) -> list[str]: ...
```

Adapter 是唯一包含 model-specific code 的地方。

## Leaderboard JSON

Schema string、timestamp、per-task scores and latency，以及 overall mean。比较 runs 时包含 per-example records，让 prediction-level regressions 可见。

## Failure modes

- Metric 返回 [0, 1] 之外的值：overall score 会变得不可解释。
- 一个 task file 中混合 metrics：assertion 会触发，每个文件只保留一种 metric。
- code_exec 没有限制 namespace：会导致任意代码执行。
- 没有 schema string：格式演进会破坏 downstream dashboards。
