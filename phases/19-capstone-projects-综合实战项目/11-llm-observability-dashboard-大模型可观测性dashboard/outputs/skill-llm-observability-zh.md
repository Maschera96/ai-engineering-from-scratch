---
name: llm-observability-zh
description: 构建一个自托管 LLM observability dashboard，摄取 OpenTelemetry GenAI spans、运行 evals，并在五分钟内捕捉 injected regressions。
version: 1.0.0
phase: 19
lesson: 11
tags: [capstone, observability, otel, langfuse, phoenix, evals, drift, clickhouse]
---

给定至少六个 SDK families（OpenAI、Anthropic、Google GenAI、LangChain、LlamaIndex、vLLM）的生产 LLM traffic，部署一个自托管 observability plane：摄取 OTLP GenAI-semconv spans、运行 evals、检测 drift 并告警。

构建计划：

1. OpenTelemetry Collector，带 OTLP HTTP receiver、tail-sampling processor（保留 100% errors、10% success、100% high-toxicity/PII），exporters 到 ClickHouse + S3。
2. ClickHouse span schema 映射 GenAI semconv：gen_ai.system、gen_ai.request.model、usage.input/output_tokens、latency_ms、user_id、app_id，再加 prompts/completions 的 JSON bag。
3. Postgres metadata store，保存 apps、users、sessions、annotation queue。
4. 每个 SDK family 各有一个 client app 使用 OpenLLMetry auto-instrumentation；验证 canonical spans 落地。
5. DeepEval + RAGAS + Phoenix evaluator pack 在 sampled traces 上定时运行；自定义 LLM-judge 处理 PII 和 off-policy。
6. 每周在 pooled prompt embeddings 上运行 PSI / KL drift detector；alert threshold 0.2。
7. Prometheus exporter 输出 eval score aggregates 和 latency percentiles；Alertmanager 到 Slack（warning）+ PagerDuty（critical）。
8. Next.js 15 App Router dashboard：overview、trace search + waterfall、eval trends、drift chart、alerts。
9. Regression probe：注入一种 1% 时间泄漏 fake SSNs 的 response pattern；度量 MTTR（alert-fire time）。

评估标准：

| Weight | Criterion | Measurement |
|:-:|---|---|
| 25 | Trace-schema coverage | 产生 canonical GenAI spans 的 SDK families 数量（目标 6+） |
| 20 | Eval 正确性 | DeepEval / RAGAS scores vs hand-labeled set |
| 20 | Dashboard UX | injected regression 上的 MTTR（目标低于 5 分钟） |
| 20 | 成本 / 规模 | 持续 1k spans/sec ingest，无 backlog |
| 15 | Alerting + drift detection | Prometheus/Alertmanager 链路端到端演练 |

硬性拒收：

- Invent 不在 OpenTelemetry GenAI semconv 中的 attribute names 的 span schemas。
- 会丢弃 errors 的 tail-sampling policies（著名 anti-pattern）。
- 按 ingest rate 运行 evals 而不 sampling（成本不可接受）。
- 只展示 “latency” 而没有区分 p50/p95/p99 的 dashboards。

拒绝规则：

- 没有 PII redaction policy 时，拒绝持久化 prompts 或 completions。
- 没有每 SDK canonical-span regression test 时，拒绝声称 “multi-SDK support”。
- 没有 baseline window 时，拒绝发布 drift detection；zero-shot drift 没用。

输出：一个仓库，包含 collector config、ClickHouse schema、Next.js 15 dashboard、eval jobs、drift detector、alerting chain、带标注 regressions 的 10k-trace demo dataset，以及一份说明文档，记录 injected PII regression 的 MTTR，并列出迭代中让 MTTR 下降的前三个 dashboard UX 改进。
