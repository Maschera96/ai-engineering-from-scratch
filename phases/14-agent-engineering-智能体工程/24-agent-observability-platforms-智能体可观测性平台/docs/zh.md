# Agent Observability: Langfuse, Phoenix, Opik

> 2026 年有三个开源智能体可观测性平台占据主导地位。Langfuse（MIT）—— 每月 600 万+ 安装量，提供追踪 + 提示管理 + 评估 + 会话回放。Arize Phoenix（Elastic 2.0）—— 深度的智能体专属评估、RAG 相关性、OpenInference 自动埋点。Comet Opik（Apache 2.0）—— 自动化提示优化、护栏、LLM-judge 幻觉检测。

**类型：** 学习
**语言：** Python (stdlib)
**前置要求：** Phase 14 · 23 (OTel GenAI)
**时长：** ~45 分钟

## 学习目标

- 说出三个顶级开源智能体可观测性平台及其许可证。
- 区分各平台最擅长的方向：Langfuse（提示管理 + 会话）、Phoenix（RAG + 自动埋点）、Opik（优化 + 护栏）。
- 解释为什么到 2026 年有 89% 的组织报告已部署智能体可观测性。
- 实现一个基于 stdlib 的「追踪到仪表盘」管线，并集成 LLM-judge 评估。

## 问题

OTel GenAI（第 23 课）给了你模式（schema）。你仍然需要一个平台来摄入 span、运行评估、存储提示版本并暴露回归。这三个竞争者各自侧重生命周期中的不同部分。

## 概念

### Langfuse (MIT)

- 每月 600 万+ SDK 安装量，19k+ GitHub stars。
- 功能：追踪、带版本控制 + playground 的提示管理、评估（LLM-as-judge、用户反馈、自定义）、会话回放。
- 2025 年 6 月：原先的商业模块（LLM-as-a-judge、标注队列、提示实验、Playground）以 MIT 许可证开源。
- 最擅长：端到端可观测性，配合紧密的提示管理闭环。

### Arize Phoenix (Elastic License 2.0)

- 更深的智能体专属评估：追踪聚类、异常检测、RAG 的检索相关性。
- 原生 OpenInference 自动埋点。
- 与托管的 Arize AX 配合用于生产环境。
- 无提示版本控制 —— 被定位为漂移/行为回归工具，与更广泛的平台搭配使用。
- 最擅长：RAG 相关性、行为漂移、异常检测。

### Comet Opik (Apache 2.0)

- 通过 A/B 实验进行自动化提示优化。
- 护栏（PII 脱敏、主题约束）。
- LLM-judge 幻觉检测。
- 来自 Comet 自家测量的基准：Opik 记录日志 + 评估耗时 23.44 秒，而 Langfuse 为 327.15 秒（约 14 倍差距）—— 厂商基准仅供方向性参考。
- 最擅长：优化闭环、自动化实验、护栏强制执行。

### 行业数据

据 Maxim（2026 年实地分析）：89% 的组织已部署智能体可观测性；质量问题是头号生产障碍（32% 的受访者提及）。

### 如何选择

| 需求 | 选择 |
|------|------|
| 集提示管理于一体的全功能方案 | Langfuse |
| 深度 RAG 评估 + 漂移 | Phoenix |
| 自动化优化 + 护栏 | Opik |
| 开放许可，非 ELv2 | Langfuse (MIT) 或 Opik (Apache 2.0) |
| Datadog / New Relic 集成 | 任意 —— 它们都导出 OTel |

### 这个模式会在哪里出错

- **没有评估策略。** 只追踪不评估只是昂贵的日志记录。
- **自建 LLM-judge 但缺乏接地（grounding）。** CRITIC 模式（第 05 课）在此适用 —— judge 需要外部工具进行事实验证。
- **提示版本未与追踪关联。** 当生产环境出现回归时，你无法二分定位到导致问题的提示。

## 动手构建

`code/main.py` 实现了一个基于 stdlib 的追踪收集器 + LLM-judge 评估器：

- 摄入 GenAI 形态的 span。
- 按会话分组，标记失败的运行（护栏触发、低置信度评估）。
- 一个脚本化的 LLM-judge，依据评分标准对智能体响应打分。
- 一个类仪表盘的摘要：失败率、首要失败原因、评估分数分布。

运行它：

```
python3 code/main.py
```

输出：每个会话的评估分数和失败分类，与 Langfuse/Phoenix/Opik 所展示的内容相匹配。

## 使用它

- **Langfuse** 自托管或云端；通过 OTel 或其 SDK 接入。
- **Arize Phoenix** 自托管；自动埋点 OpenInference。
- **Comet Opik** 自托管或云端；自动化优化闭环。
- **Datadog LLM Observability** 适用于已经在运行 Datadog 的混合 ops+ML 团队。

## 交付它

`outputs/skill-obs-platform-wiring.md` 选定一个平台，并将追踪 + 评估 + 提示版本接入现有智能体。

## 练习

1. 将一周的 OTel 追踪导出到 Langfuse 云端（免费层）。哪些会话失败了？为什么？
2. 为你的领域编写一个 LLM-judge 评分标准（事实正确性、语气、范围遵循度）。在 50 条追踪上测试。
3. 对比 Langfuse 的提示版本控制与 Phoenix 的追踪聚类。哪个能更快告诉你哪里坏了？
4. 阅读 Opik 的护栏文档。为你的某次智能体运行接入一个 PII 脱敏护栏。
5. 在你自己的语料上对三者进行基准测试。忽略厂商公布的数字；测量你自己的。

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| Tracing | "Span 收集器" | 摄入 OTel / SDK span；按会话建索引 |
| Prompt management | "提示 CMS" | 与追踪关联的带版本提示 |
| LLM-as-judge | "自动化评估" | 独立的 LLM 依据评分标准对智能体输出打分 |
| Session replay | "追踪回放" | 逐步回溯过去的运行以便调试 |
| RAG relevancy | "检索质量" | 检索到的上下文是否与查询匹配 |
| Trace clustering | "行为分组" | 聚类相似的运行以检测漂移 |
| Guardrail enforcement | "日志时策略" | 对记录内容进行 PII/毒性/范围检查 |

## 延伸阅读

- [Langfuse docs](https://langfuse.com/) —— 追踪、评估、提示管理
- [Arize Phoenix docs](https://docs.arize.com/phoenix) —— 自动埋点、漂移
- [Comet Opik](https://www.comet.com/site/products/opik/) —— 优化 + 护栏
- [OpenTelemetry GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) —— 三者都消费的模式
