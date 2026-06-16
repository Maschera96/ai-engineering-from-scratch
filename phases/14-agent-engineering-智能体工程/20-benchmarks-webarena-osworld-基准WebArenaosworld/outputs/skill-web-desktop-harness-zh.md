---
name: web-desktop-harness-zh
description: 构建 WebArena/OSWorld 风格的测试框架，提供基于执行的评估与轨迹效率指标。
version: 1.0.0
phase: 14
lesson: 20
tags: [webarena, osworld, harness, trajectory-efficiency]
---

给定一个目标应用（Web 或桌面）以及一组带有黄金轨迹的任务，构建一个评估测试框架。

产出：

1. 任务定义：`(tid, description, gold_steps, success_predicate, state_reset)`。
2. 运行器：运行智能体，捕获每一个动作，记录步数 + 耗时 + 成功状态。
3. 轨迹效率指标：`agent_steps / gold_steps`。按任务报告并汇总。
4. 任务之间重置状态——绝不要在被另一个任务弄脏的状态上运行某个任务。
5. 失败模式分类器：对每次失败，标记它是接地失误（选错元素）还是规划失误（选错动作）。

硬性拒绝：

- 任务之间不重置状态。跨任务污染会使所有分数失效。
- 仅报告成功率。轨迹效率才是 2026 年的标准。
- 仅有截图、缺乏 DOM 对等的测试框架。有些智能体同时使用 DOM+视觉；除非专门限制操作面，否则两者都要提供。

拒绝规则：

- 如果任务没有黄金轨迹，拒绝。没有它们你无法衡量效率。
- 如果应用没有固定到某个具体版本，拒绝。漂移会使跨运行比较失效。
- 如果智能体拥有破坏性工具（删除、发布），要求使用应用的沙箱副本。

输出：`tasks.py`、`runner.py`、`failure_classifier.py`、`report.py`、`README.md`，说明重置策略、黄金轨迹来源，以及接地与规划的划分。结尾给出"接下来读什么"，指向第 21 课（计算机使用模型）或第 30 课（评估驱动开发）。
