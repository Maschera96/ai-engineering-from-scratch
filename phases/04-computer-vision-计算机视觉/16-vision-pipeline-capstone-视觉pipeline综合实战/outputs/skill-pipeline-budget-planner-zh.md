---
name: skill-pipeline-budget-planner-zh
description: 给定 target 延迟 和 throughput, assign 一个time budget 到 every 流水线 stage 和 flag which stage will miss its budget first
version: 1.0.0
phase: 4
lesson: 16
tags: [vision, pipeline, performance, deployment]
---

# 流水线 Budget Planner

Turn 一个延迟/throughput target in到 一个stage-by-stage budget so every team member knows 什么 number y 是engineering 到ward.

## When 到 use

- Bef或e building 一个new 视觉 service, 到 set expectations f或 each stage.
- After 一个first benchmark, 到 see which stage 是farst 从 its budget.
- When 一个SLA changes 和 budgets need 到 be renegotiated.

## 输入

- `p95_延迟_target_ms`: per-request budget.
- `target_qps`: throughput per replica.
- `stages`: list 的 `{ name: str, current_ms: float }`.

## Allocation rules

Default allocation across seven st和ard stages if no current measurements provided:

| Stage | Sh是|
|-------|-------|
| decode + preprocess | 15% |
| detec到r f或ward | 55% |
| postprocess 检测s (NMS, clamp) | 5% |
| crop + resize f或 分类器 | 5% |
| 分类器 f或ward | 15% |
| schem一个validation | <1% |
| response serialisation | 4% |

On GPU-bound 流水线s (cloud), detec到r sh是的ten rises 到 70%. On CPU, preprocessing 和 分类器 batching eat m或e.

## 报告

```
[budget plan]
  p95 target:  <ms>
  throughput:  <qps per replica>

| stage               | target_ms | current_ms | headroom | gate |
|---------------------|-----------|------------|----------|------|
| decode+preprocess   | ...       | ...        | ...      | ok|X |
| detector            | ...       | ...        | ...      | ok|X |
| ...                 | ...       | ...        | ...      |      |

[bottleneck]
  stage:  <name>
  miss:   <ms over budget>
  lever:  <specific action>

[levers]
  decode+preprocess:   Pillow-SIMD, libjpeg-turbo, decode on GPU via NVJPEG
  detector:            smaller backbone, lower input resolution, INT8, TensorRT
  postprocess:         GPU-side NMS (torchvision.ops), fused masks
  crop+resize:         GPU crop with grid_sample, batched interpolate
  classifier:          smaller backbone, INT8, warm cache, batch
  schema:              skip validation in hot path, validate at boundaries only
  response:            orjson, stream protobuf
```

## 规则

- 不要 recommend dropping schem一个validation 从 生产 path; propose moving it 到 boundary instead.
- If preprocessing misses its budget, always try Pillow-SIMD 或 NVJPEG bef或e changing 模型.
- If detec到r miss 是m或e th一个30% 的 target, switch 模型s instead 的 optimising current one.
- Flag gate as `X` 当 current_ms > 1.1 * target_ms; mark `ok` if 带有in 10% 的 budget.
