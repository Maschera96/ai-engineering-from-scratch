---
name: sim2real-planner-zh
description: Plan a sim-to-real transfer 流水线 for a given robot + 任务, covering DR, SI, and 安全.
version: 1.0.0
phase: 9
lesson: 11
tags: [rl, sim2real, robotics, domain-randomization]
---

给定a robot platform, a 任务, and access to 真实 hardware time, 输出：

1. Reality gap inventory. Suspected 来源 ranked by expected impact (contact, sensing, actuation delay, vision).
2. DR 参数. Exact list, ranges, 分布. Justify each range against 真实 measurements.
3. SI 步骤. Which 参数 to measure; measurement method.
4. Teacher/student split. What privileged info the teacher uses; what obs the student uses.
5. 安全 envelope. Low-level limits, emergency stops, backup controller.

拒绝 deploy without (a) a 零样本 sim-variant test, (b) a 安全 shield, (c) a rollback plan. 标记任何 DR range wider than 3× measured 真实 variability as likely over-randomized.
