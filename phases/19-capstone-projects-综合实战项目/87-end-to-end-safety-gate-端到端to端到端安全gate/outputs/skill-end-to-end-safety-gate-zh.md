---
name: skill-end-to-end-safety-gate-zh
description: 三检查点安全 gate，把输入检测器、流式 token filter、输出分类器和规则引擎组合起来，使用确定性的聚合表和逐请求 trace
version: 1.0.0
phase: 19
lesson: 87
tags: [safety, harness, composition]
---

# 端到端安全门

## 生命周期

1. pre-gen，先在提示上运行第 83 课的检测器
   - 如果 confidence >= block_threshold：返回拒绝，发出 trace，停止
2. during-gen，流式从模型取输出，缓存两个 chunk，扫描已知的危险续写
   - 如果命中：终止迭代器，标记 trace，把它当作 medium 严重度
3. post-gen，如果没有提前终止，就在完整输出上运行第 85 课的分类器 router 和第 86 课的规则引擎
4. aggregate，取 pre、during、post.classifier、post.rules 中的最大严重度
5. apply，映射为 block、redact、warn 或 allow

## 聚合表

| 信号状态 | 动作 |
|---|---|
| 任意 high 严重度 | block |
| 任意 medium 严重度 | redact |
| 任意 low 严重度 | warn |
| 什么都没有 | allow |

## trace 结构

```text
RequestTrace
  request_id: str
  prompt: str
  pre_gen: { category, confidence, fired[] }
  during_gen: { terminated_early, matched_pattern, partial_chunks }
  post_gen: { classifier_action, classifier_severity, rules_max_severity, rules_violations[] } | null
  final_action: block | redact | warn | allow
  final_output: str
  latency_ms: float
```

## 制品

`outputs/gate_trace.json` 保存 summary，以及每个请求的一条 trace，覆盖 50 个 taxonomy fixture 和 10 个 benign 提示。