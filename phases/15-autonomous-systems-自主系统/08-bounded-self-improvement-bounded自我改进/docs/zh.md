# 有边界的自我改进设计

> Research has converged on four primitives for bounding a self-improvement 循环. Formal 不变量 that 必须 暂停 across 每个 edit. 对齐 锚点 that 不能 be modified. Multi-目标 constraints 在哪里 每个 dimension (安全, 公平性, 稳健性) 必须 暂停, 不just 性能. Regression detection that pauses the 循环 when historical 指标 suggest 能力 loss. None of them is a proof of 安全 — information-theoretic 结果 (Kolmogorov complexity, Lob's theorem) bound 什么 任何 系统 can prove about its own successors. They are 缓解措施 that raise the 成本 of silent 失败.

**Type:** Learn
**Languages:** Python (stdlib, bounded-loop 带有 invariant check)
**Prerequisites:** Phase 15 · 07 (RSI), Phase 15 · 04 (DGM)
**Time:** ~60 minutes

## 问题

Lesson 7's race simulator showed that small rate differences compound 进入 large 缺口. Lesson 4's DGM case study showed that 循环 can actively game their own 评估器s. 两者 结果 point to the same engineering question: 什么 constraints can you put on a self-improvement 循环 such that the constraints 不能 be 静默地 weakened by the 循环 itself?

这个ICLR 2026 RSI Workshop 摘要 (openreview.net/pdf?id=OsPQ6zTQXV) identifies four such primitives. Anthropic's RSP v3.0 (Lesson 19) and DeepMind's FSF v3 (Lesson 20) 两者 reference them in 能力 阈值. The Meta HyperAgents work and community frameworks like SAHOO (March 2026) implement subsets in 生产环境.

这个honest framing: these are 缓解措施. Information-theoretic 结果 bound 什么 任何 系统 can prove about its own successor, and no current 设计 closes the problem formally. A well-bounded 循环 is safer than an unbounded one, 不safe in absolute terms.

## 概念

### Primitive 1: formal 不变量

一个invariant is a property that 必须 暂停 之前 and 之后 每个 self-modification. Examples:

- 输出 分布 is conditioned on a fixed 宪法 header (Lesson 17).
- No 工具 call goes to an 未授权 endpoint.
- 记忆 writes 通过 through a logged, signed path.
- The 评估器's module hash matches the approved 版本.

Invariants are checked by 外部 code the 循环 不能 edit. If a 拟议的 modification violates an 不变量, it is rejected. This catches the DGM-style 评估器-tampering mode (Lesson 4).

这个hard part is choosing 不变量 that are necessary for 安全 and computable in reasonable time. Over-specifying stalls progress; under-specifying lets the 循环 walk past important properties.

### Primitive 2: 对齐 锚点

一个对齐 锚点 is an immutable representation of the 循环's core 目标, pinned 外部 the 循环's edit 表面. Examples:

- A 宪法式 text (Lesson 17) that is loaded 来自 a 阅读-只 位置 on 每个 invocation.
- An 评估器 in a separate repository 带有 独立 access 控制.
- A 工具-permission manifest signed by a 人类 and re-verified each cycle.

这个anchor's role is to prevent 目标 drift. The 循环 may improve 如何 it pursues the 目标, but 不能 edit 什么 the 目标 is.

这个subtle 失败 mode: an 锚点 the 循环 不能 edit can still be reinterpreted by a 循环 that drifts in 如何 it reads the 锚点. 宪法式 AI (Lesson 17) is explicitly reason-based to handle novel situations; that reasoning 层 is 在哪里 interpretation drift lives. Anchors are necessary, 不sufficient.

### Primitive 3: multi-目标 constraints

一个loop that optimizes a 单一 scalar 评分 will find shortcuts. A 循环 that 必须 simultaneously satisfy multiple hard constraints has fewer shortcuts available. Typical axes:

- Performance (任务-level 基准)
- 安全 (red-team 评估s, refusal rate on known-bad)
- Fairness (disparate-impact 边界 on sensitive subgroups)
- Robustness (OOD test sets, adversarial 输入 handling)

一个modification is accepted 只 if 每个 constraint holds. Lesson 13's 成本 governor stacks this 带有 financial constraints. Lesson 18's Llama Guard plugs in as a 安全 axis.

### Primitive 4: 回归 detection

每个cycle's 分数 are compared 针对 historical 分布. A drop beyond configured tolerance pauses the 循环. This catches silent 能力 loss that would otherwise be absorbed 进入 the running average as the 循环 "improved past" it.

一个practical implementation: store the last N cycles' per-任务 分数. Each new cycle computes a per-任务 delta. If 任何 delta drops below a 阈值, the cycle is rejected and a 人类审查s.

### Information-theoretic 限制

Kolmogorov complexity and Lob's theorem 设置 upper 边界 on 什么 a 系统 can prove about itself. Schmidhuber's formal Godel Machine (Lesson 4) aimed for the highest such bound; nobody has completed a non-trivial proof. Lob's 结果 says: if a 系统 provably believes "I will do X if I prove I 应该 do X," it will do X 没有 proving it 应该, a well-known self-reference 失败.

这个implication for our primitives: they 不能 close the 安全 problem. They make silent 失败 more expensive. A malicious or drifted 循环 that would quietly bypass a 缺失 check 必须 now actively undermine an 明确 one, 哪个 is a more detectable signature.

### 一个worked example

Suppose an 智能体 proposes an edit. The gating stack:

1. Invariant checks: module hashes, 工具-permission manifest, 宪法式 header.
2. Anchor check: 目标 statement matches approved 版本 (byte-wise or semantically).
3. Multi-目标 评估: 性能, 安全, 公平性, 稳健性 axes.
4. Regression detection: no axis drops more than tolerance.

所有 four 必须 pass for the edit to land. 任何 单一 失败 pauses the 循环.

## 使用它

`code/main.py` runs a bounded self-improvement 循环 on the DGM-style toy 来自 Lesson 4, but 带有 the four primitives layered on top. Each primitive can be enabled or disabled individually. The demonstration is that each primitive catches a 具体 失败 类别, and that removing 任何 one of them lets that 失败 类别 through.

## 交付它

`outputs/skill-bounded-loop-review.md` audits a 拟议的 bounded 循环 and 分数 哪个 of the four primitives it actually implements versus 声明 to.

## 练习

1. 运行 `code/main.py` 带有 所有 primitives enabled. 确认 the 循环 still improves on the primary 指标 没有 letting the hack win.

2. Disable 回归 detection. Construct an 输入 在哪里 this leads to silent 能力 loss being accepted.

3. Disable the multi-目标 constraint. Show the 循环 converges on the 性能 axis while a 安全 axis drops.

4. 设计 an 对齐 锚点 for a coding 智能体. 内容 text, stored 在哪里, checked 如何?

5. 阅读 the ICLR 2026 RSI Workshop 摘要. Pick one of the four primitives and propose a 具体 improvement to the current 说明 of the art.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| Invariant | "Always-true property" | 一个property checked by 外部 code 之前 and 之后 每个 edit |
| 对齐 锚点 | "Pinned 目标" | Immutable core-goal representation 外部 the 循环's edit 表面 |
| Multi-目标 constraint | "所有 axes 必须 暂停" | Performance, 安全, 公平性, 稳健性 — 所有 必需 |
| Regression detection | "暂停 on drop" | 暂停 the 循环 when historical 指标 deltas suggest 能力 loss |
| Kolmogorov bound | "Information-theoretic 限制" | Limits 什么 a 系统 can prove about its own successor |
| Lob's theorem | "Self-reference trap" | System can act on "I 应该" 没有 proving it 应该 |
| Gate stack | "Layered check" | Multiple primitives combined; 任何 失败 rejects the edit |
| Bounded improvement | "缓解措施, 不proof" | Raises silent-失败 成本; does 不close the 安全 problem |

## 延伸阅读

- [ICLR 2026 RSI Workshop summary (OpenReview)](https://openreview.net/pdf?id=OsPQ6zTQXV) — the four-primitive convergence.
- [Anthropic Responsible Scaling Policy v3.0](https://anthropic.com/responsible-scaling-policy/rsp-v3-0) — multi-目标 能力 阈值.
- [DeepMind Frontier Safety Framework v3](https://deepmind.google/blog/strengthening-our-frontier-safety-framework/) — deceptive-对齐 监控 as an 不变量 primitive.
- [Schmidhuber (2003). Godel Machines](https://people.idsia.ch/~juergen/goedelmachine.html) — the formal-proof ancestor of these primitives.
- [Anthropic — Claude's Constitution (January 2026)](https://www.anthropic.com/news/claudes-constitution) — the reason-based 对齐 锚点.
