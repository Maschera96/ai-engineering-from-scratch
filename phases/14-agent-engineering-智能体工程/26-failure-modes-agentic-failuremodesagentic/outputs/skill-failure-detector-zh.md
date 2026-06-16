---
name: failure-detector-zh
description: 为智能体 trace 生成失败模式检测器，接入 trace 存储，标记五种行业中反复出现的模式以及领域特定的特征签名。
version: 1.0.0
phase: 14
lesson: 26
tags: [failure-modes, masft, detection, observability]
---

给定一个产品领域和一个 trace 存储，生成针对智能体失败模式的检测器。

产出：

1. 每种模式对应一个检测器：`hallucinated_action`、`scope_creep`、`cascading_errors`、`context_loss`、`tool_misuse`、`success_hallucination`。
2. 领域特定的检测器（例如，对于开发工具是“创建了 PR 却没有关联 issue”，对于营销工具是“在没有确认的情况下向 > 5 个收件人发送了邮件”）。
3. 标记器，对每个 trace 应用所有检测器并输出一个分布。
4. 基于阈值的告警：如果今天 >=5% 的 trace 被标记为某种模式，则呼叫值班人员或开一个工单。
5. 样本保留：对每个被标记的 trace，保留输入 + 输出 + 状态快照，供运维人员审查。

硬性拒绝：

- 在生产中需要对每个 trace 进行 LLM 调用的检测器。使用基于模式的检测器；将 LLM-judge 保留给抽样审查。
- 仅在崩溃时标记。大多数失败会产出看起来有效的输出。必须对内容 + 状态进行签名检查。
- 在未做 PII 脱敏的情况下存储被标记的 trace。失败样本携带着最糟糕的内容；存储前先清洗。

拒绝规则：

- 如果用户想要“永久存储所有 trace”，出于成本 + 合规原因拒绝。按标签 + 比率抽样。
- 如果产品没有“已知良好”的基线，拒绝漂移告警。漂移需要一个参考。
- 如果检测器没有版本化，拒绝。检测器回退会在你毫无察觉的情况下破坏你的信号。

输出：`detectors.py`、`tagger.py`、`alerts.py`、`retention.py`、`README.md`，解释阈值、保留策略、告警路由。以“接下来读什么”结尾，指向 Lesson 24（可观测性后端）或 Lesson 27（提示注入）以了解对抗性失败模式。
