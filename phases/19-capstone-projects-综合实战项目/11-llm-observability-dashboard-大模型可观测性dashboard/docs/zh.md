# Capstone 11 — LLM 可观测性与评测 Dashboard

> Langfuse 转向了 open-core。Arize Phoenix 发布了 2026 年 GenAI semconv 映射。Helicone 和 Braintrust 都进一步强化了按用户归因成本。Traceloop 的 OpenLLMetry 成了事实上的 SDK instrumentation 标准。2026 年生产环境的典型形态是，ClickHouse 存 traces，Postgres 存 metadata，Next.js 做 UI，再配上一小支对抽样 traces 运行 eval jobs 的队伍，包含 DeepEval、RAGAS 和 LLM-judge。构建一个自托管版本，至少从四个 SDK 家族摄取数据，并演示在五分钟内捕捉到一次人为注入的回归。

**Type:** Capstone
**Languages:** TypeScript (UI), Python / TypeScript (ingest + evals), SQL (ClickHouse)
**Prerequisites:** Phase 11 (LLM engineering), Phase 13 (tools), Phase 17 (infrastructure), Phase 18 (safety)
**Phases exercised:** P11 · P13 · P17 · P18
**Time:** 25 hours

## Problem

到 2026 年，任何在生产环境跑 AI 流量的团队，都会在模型旁边维护一层可观测性平面。成本归因。幻觉检测。漂移监控。越狱信号。SLO dashboard。PII 泄漏告警。开源参考方案，Langfuse、Phoenix、OpenLLMetry，已经在 OpenTelemetry GenAI semantic conventions 上收敛，把它作为摄取 schema。现在你可以用一个 SDK 给 OpenAI、Anthropic、Google、LangChain、LlamaIndex 和 vLLM 做 instrumentation，并发送兼容的 spans。

你将构建一个自托管 dashboard，能从至少四个 SDK 家族摄取数据，对抽样 traces 运行一小组 eval jobs，检测 drift，并发出告警。衡量标准是，给定一个刻意注入的回归，比如某个 prompt 开始产出 PII，这个 dashboard 能在五分钟内发现并触发告警。

## Concept

摄取入口是 OTLP HTTP。SDK 会产出 GenAI-semconv spans，例如 `gen_ai.system`、`gen_ai.request.model`、`gen_ai.usage.input_tokens`、`gen_ai.response.id`、`llm.prompts`、`llm.completions`。Spans 落到 ClickHouse，用于列式分析。metadata，比如 users、sessions、apps，则落到 Postgres。

Evals 以批处理作业的形式在抽样 traces 上运行。DeepEval 评分 faithfuless、toxicity 和 answer relevance。若 trace 携带 retrieval context，RAGAS 会给 retrieval metrics 打分。自定义 LLM-judges 则负责运行领域专属检查，比如 PII 泄漏、off-policy response。Eval runs 会回写到同一个 ClickHouse，作为与父 trace 关联的 eval spans。

Drift detection 监控随时间变化的 embedding 空间分布，方法包括 prompt embeddings 上的 PSI 或 KL divergence，同时也看 eval score 的趋势。Alerts 会送到 Prometheus Alertmanager，再转发到 Slack / PagerDuty。UI 用的是带 Recharts 的 Next.js 15。

## Architecture

```
production apps:
  OpenAI SDK  +  Anthropic SDK  +  Google GenAI SDK
  LangChain + LlamaIndex + vLLM
       |
       v
  OpenTelemetry SDK with GenAI semconv
       |
       v  OTLP HTTP
  collector (ingest, sample, fan-out)
       |
       +-------------+-----------+
       v             v           v
   ClickHouse    Postgres    S3 archive
   (spans)       (metadata)  (raw events)
       |
       +---> eval jobs (DeepEval, RAGAS, LLM-judge)
       |     sampled or all-trace
       |     write eval spans back
       |
       +---> drift detector (PSI / KL on prompt embeddings)
       |
       +---> Prometheus metrics -> Alertmanager -> Slack / PagerDuty
       |
       v
   Next.js 15 dashboard (Recharts)
```

## Stack

- Ingest: OpenTelemetry SDKs + GenAI semantic conventions，OTLP HTTP transport
- Collector: OpenTelemetry Collector，带 tail-sampling processor，用于成本控制
- Storage: ClickHouse 存 spans，Postgres 存 metadata，S3 存原始事件归档
- Evals: DeepEval、RAGAS 0.2、Arize Phoenix evaluator pack、自定义 LLM-judge
- Drift: 每周在聚合后的 prompt embeddings（sentence-transformers）上运行 PSI / KL
- Alerting: Prometheus Alertmanager -> Slack / PagerDuty
- UI: Next.js 15 App Router + Recharts + server actions
- 开箱即用支持的 SDKs: OpenAI、Anthropic、Google GenAI、LangChain、LlamaIndex、vLLM

## Build It

1. **Collector config.** 配置 OpenTelemetry Collector，启用 OTLP HTTP receiver、tail-sampler，对 errored traces 保留 100%，对成功请求保留 10%，并把数据导出到 ClickHouse 和 S3。

2. **ClickHouse schema.** 创建表 `spans`，列与 GenAI semconv 对齐，包括 `gen_ai_system`、`gen_ai_request_model`、`input_tokens`、`output_tokens`、`latency_ms`、`prompt_hash`、`trace_id`、`parent_span_id`，并用一个 JSON bag 存较长 payload。为 `user_id` 和 `app_id` 增加二级索引。

3. **SDK coverage test.** 为每个 SDK 写一个小型 client app，覆盖 OpenAI、Anthropic、Google、LangChain、LlamaIndex、vLLM，并使用 OpenLLMetry auto-instrument。验证每个 SDK 都能产出标准的 GenAI spans，并成功落到 ClickHouse。

4. **Eval jobs.** 定时作业读取最近 15 分钟的抽样 traces，运行 DeepEval 的 faithfulness、toxicity 和 answer relevance。输出结果作为与父 trace 关联的 eval spans。

5. **Custom LLM-judge.** 实现一个 PII-leak judge。给定一条 response，调用一个 guard LLM，为其 PII 泄漏可能性打分。高分 response 进入 triage queue。

6. **Drift detection.** 每周作业计算本周聚合 prompt embeddings 与过去连续 4 周 baseline 之间的 PSI。若 PSI 高于阈值则告警。

7. **Dashboard.** 用 Next.js 15 构建页面，包括 overview（spans/sec、cost/user、p95 latency）、traces（search + waterfall）、evals（faithfulness trend、toxicity）、drift（PSI over time）和 alerts。

8. **Alerting chain.** Prometheus exporter 读取 eval score 聚合和 latency percentiles。Alertmanager 将 warnings 路由到 Slack，将 critical breaches 路由到 PagerDuty。

9. **Regression probe.** 注入一个 bug，让被评估的 chatbot 有 1% 的概率泄漏假的 SSN。测量 MTTR，也就是从 bug 部署到 Slack 告警之间的时间。

## Use It

```
$ curl -X POST https://my-otel-collector/v1/traces -d @trace.json
[collector]  accepted 1 trace, 3 spans
[clickhouse] inserted 3 spans (app=chat, user=u_42)
[eval]       DeepEval faithfulness 0.82, toxicity 0.03
[drift]      weekly PSI 0.08 (below 0.2 threshold)
[ui]         live at https://obs.example.com
```

## Ship It

`outputs/skill-llm-observability.md` 是可交付成果。给定一个 LLM 应用，这个 dashboard 会摄取它的 traces，运行 evals，对 drift 告警，并在 Next.js 中展示按用户拆分的成本明细。

| Weight | Criterion | How it is measured |
|:-:|---|---|
| 25 | Trace-schema coverage | 能产出标准 GenAI spans 的 SDK 家族数量，目标为 6+ |
| 20 | Eval correctness | DeepEval / RAGAS scores 与人工标注集合对比 |
| 20 | Dashboard UX | 注入回归时的 MTTR，目标低于 5 分钟 |
| 20 | Cost / scale | 在无 backlog 的情况下持续摄取 1k spans/sec |
| 15 | Alerting + drift detection | Prometheus/Alertmanager 链路完成端到端演练 |
| **100** | | |

## Exercises

1. 为 Haystack framework 增加自定义 instrumentation。验证标准 spans 能落到 ClickHouse，并携带准确的 `gen_ai.*` attributes。

2. 在同一批 traces 上，用 Phoenix evaluators 替换 DeepEval。测量两种 eval engine 之间的 score drift。

3. 强化 drift detector，不再全局计算 PSI，而是按 app-id 分开计算。展示按应用拆分的 drift 轨迹。

4. 增加一个 “user impact” 页面，展示每用户成本和每用户失败率，并配上 sparklines。

5. 构建一个 tail-sampling policy，保留 100% 的 toxicity > 0.5 traces，并对其余 traces 做 10% 的分层抽样。测量由此引入的 sampling bias。

## Key Terms

| Term | What people say | What it actually means |
|------|-----------------|------------------------|
| GenAI semconv | “OTel LLM attributes” | 2025 年 OpenTelemetry 关于 LLM span attributes 的规范，覆盖 system、model、tokens |
| Tail sampling | “Post-trace sample” | Collector 在 trace 完成后决定保留还是丢弃它，因此可以查看错误信息 |
| PSI | “Population stability index” | 比较两个分布的 drift 指标，通常 > 0.2 代表有意义的漂移 |
| LLM-judge | “Eval as model” | 用一个 LLM 按 rubric 给另一个 LLM 的输出打分，比如 faithfulness、toxicity、PII |
| Tail-sampling policy | “Keep-rule” | 决定哪些 traces 持久化、哪些丢弃的规则，通常包含错误保留和采样率 |
| Eval span | “Linked eval trace” | 携带 eval score 的子 span，并与原始 LLM 调用 span 关联 |
| Cost per user | “Unit economics” | 在某个时间窗口内归因到 `user_id` 的美元成本，是关键产品指标 |

## Further Reading

- [Langfuse](https://github.com/langfuse/langfuse) — 参考性的 open-core observability 平台
- [Arize Phoenix](https://github.com/Arize-ai/phoenix) — 另一个参考方案，drift 支持很强
- [OpenLLMetry (Traceloop)](https://github.com/traceloop/openllmetry) — auto-instrumentation SDK 家族
- [OpenTelemetry GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — 摄取 schema
- [Helicone](https://www.helicone.ai) — 另一种托管 observability 方案
- [Braintrust](https://www.braintrust.dev) — 另一种以 eval 为先的平台
- [ClickHouse documentation](https://clickhouse.com/docs) — 列式 span 存储文档
- [DeepEval](https://github.com/confident-ai/deepeval) — evaluator library
