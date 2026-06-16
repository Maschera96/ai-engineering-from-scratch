---
name: obs-platform-wiring-zh
description: 选择一个可观测性平台（Langfuse、Phoenix、Opik、Datadog），并将 traces + evals + prompt 版本接入到现有智能体中。
version: 1.0.0
phase: 14
lesson: 24
tags: [observability, langfuse, phoenix, opik, datadog, tracing]
---

给定一个智能体运行时和产品需求，选择一个可观测性平台并搭建接入骨架。

决策：

1. 需要在同一处管理 prompt + 会话回放 -> **Langfuse**。
2. 需要深度 RAG 相关性 + 漂移/异常检测 -> **Phoenix**。
3. 需要自动化 prompt 优化 + PII 护栏 -> **Opik**。
4. 已经在运行 Datadog -> **Datadog LLM Observability**（从 v1.37+ 起原生映射 GenAI）。
5. 需要无 ELv2 许可证 -> **Langfuse**（MIT）或 **Opik**（Apache 2.0）；纯 OSS 分发场景下避免使用 Phoenix。

产出：

1. OTel GenAI instrumentation（第 23 课）—— 这是通用底座。
2. 平台专属的 SDK 或 OTel exporter 配置。
3. 面向你领域的 LLM-judge 评分标准（事实正确性、范围、语气、拒答质量）。
4. 将 prompt 版本管理接入 traces（Langfuse），或 trace 聚类配置（Phoenix），或实验定义（Opik）。
5. 对所记录内容的护栏：PII 脱敏、密钥清除。
6. 仪表盘：会话健康度、失败分类、延迟分布、每会话成本。

硬性拒绝：

- 不带 evals 就上线。仅有 tracing 只是昂贵的日志记录。
- 使用没有外部验证的自写 LLM-judge。CRITIC 模式（第 05 课）：judge 需要外部工具来做事实接地。
- 在 span body 中存储 PII。始终使用外部存储 + 引用 ID。

拒绝规则：

- 如果用户要求“一个平台搞定一切”，拒绝并提供上述决策。没有任何单一平台能在三个维度上全面占优。
- 如果产品对每个智能体任务都没有验收标准，拒绝交付 evals。LLM-judge 需要评分标准；评分标准需要产品决策。
- 如果用户想要“不采样，全量捕获”，拒绝。trace 量随流量线性增长；规模化时必须做采样（head-based 或 tail-based）。

输出：`instrumentation.py`、`judge.py`、`dashboards.md`、`README.md`，说明平台选择、评分标准、采样策略和事件响应。最后以“接下来读什么”结尾，指向第 30 课（eval 驱动开发）或第 26 课（失败模式分类）。
