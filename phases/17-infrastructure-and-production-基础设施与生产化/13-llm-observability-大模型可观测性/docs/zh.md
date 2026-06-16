# 大模型可观测性栈选择

> The 2026 可观测性 market splits into two categories. Development 平台 (LangSmith, Langfuse, Comet Opik) bundle 监控 搭配 评估, 提示词 management, session replays. 网关/埋点 tools (Helicone, SigNoz, OpenLLMetry, Phoenix) focus on telemetry. Langfuse is MIT-licensed core with strong OSS balance (50K events/月 free cloud). Phoenix is OpenTelemetry-native under Elastic License 2.0 ： excellent for drift/RAG visualization, not a persistent 生产化 backend. Arize AX uses zero-copy Iceberg/Parquet integration claiming 100x 更便宜 than monolithic 可观测性. LangSmith leads for LangChain/LangGraph, $39/user/mo, self-host in 企业级 only. Helicone is proxy-based with 15-30 min setup, 100K req/mo free, but less depth on agent 追踪. 常见生产化 模式: 网关 (Helicone/Portkey) + eval 平台 (Phoenix/TruLens) glued by OpenTelemetry.

**Type:** Learn
**Languages:** Python（标准库， 玩具 trace-sampling模拟器)
**Prerequisites:** Phase 17 · 08 (Inference Metrics), Phase 14 (智能体工程)
**Time:** ~60 分钟

## 学习目标

- Distinguish development 平台 (bundled: 评估 + 提示词 + sessions) from 网关/telemetry tools (追踪 + 指标 only).
- Map six major tools (Langfuse, LangSmith, Phoenix, Arize AX, Helicone, Opik) to their licensing, 定价, and sweet-spot use cases.
- 解释 the OpenTelemetry-glue 模式 that lets you combine 一个网关 tool with a separate eval 平台.
- Name the 2026 成本 差异点 (Arize AX's zero-copy approach 对比 monolithic ingest) and state the rough 100x multiplier.

## 问题

You shipped an LLM 功能. It works. You have no visibility into 提示词 failures, tool loops, 延迟 regressions, 成本 spikes, 或 提示词-cache hit rate. You Google "LLM 可观测性" and get eight tools all claiming they solve the same problem at three different price points.

They don't solve the same problem. LangSmith answers "why did this LangGraph run fail?" Phoenix answers "is my RAG pipeline drifting?" Helicone answers "which app is burning 词元?" Langfuse answers "can I self-host the whole thing?" Different tools, different audiences.

Picking involves four axes: stack (LangChain? raw SDK? multi-vendor?), license tolerance (MIT only? Elastic OK? commercial fine?), budget (free tier? $100/mo? $1000/mo?), and self-host (must? nice-to-have? never?).

## 概念

### 两类

**Development 平台** bundle 可观测性 搭配 评估, 提示词 management, dataset versioning, session replay. You run experiments, see which 提示词 worked, dataset-regression a new 提示词 against old winners. LangSmith, Langfuse, Comet Opik.

**网关/telemetry tools** instrument inference calls ： 提示词, response, 词元, 延迟, 模型, 成本. Helicone, SigNoz, OpenLLMetry, Phoenix. Minimalist. Can be combined with a separate eval tool via OpenTelemetry.

### Langfuse ： OSS balance

- Core Apache / MIT licensed; self-host via Docker.
- Cloud free tier: 50K events/月. Paid: $29/mo for 团队.
- 评估, 提示词 management, 追踪, datasets. Reasonable coverage of all four dev-平台 功能.
- Sweet spot: you want LangSmith-class 功能 but must self-host or stay on OSS license.

### Phoenix (Arize) ： telemetry-first, OpenTelemetry-native

- Elastic License 2.0; self-host trivial.
- Excellent at RAG and drift visualization. Embedding-space scatter plots shipped as first-class.
- Not designed as persistent 生产化 backend ： primarily development-time 可观测性.
- Sweet spot: RAG pipeline development, drift debugging, pairs with a separate 网关 for 生产化.

### Arize AX ： the scale play

- Commercial. Zero-copy data lake integration via Iceberg/Parquet.
- Claims ~100x 更便宜 than monolithic 可观测性 (Datadog-class) at scale. The math: you store 追踪 in your own Parquet on S3; Arize reads 直接.
- Sweet spot: >10M 追踪/day, existing data lake, want LLM-具体 dashboards without Datadog 定价.

### LangSmith ： LangChain/LangGraph first

- Commercial, $39/user/月. Self-host only on 企业级.
- Best-in-class for LangChain and LangGraph stacks. If you are not on either, it is less compelling.
- Sweet spot: 团队 committed to LangChain, willing to pay.

### Helicone ： proxy-based minimum viable

- 15-30 minute setup by swapping your `OPENAI_API_BASE` to Helicone proxy.
- MIT licensed; 100K req/mo free, paid $20/mo+.
- Includes 故障切换, caching, 限流 ： acts as 一个网关 too.
- Less depth on agent / multi-步骤 追踪.
- Sweet spot: quick start, single-stack app, need 网关 + 可观测性 in one.

### Opik (Comet) ： OSS dev 平台

- Apache 2.0, fully OSS.
- 类似 功能 set to Langfuse with Comet heritage.
- Sweet spot: ML 团队 already on Comet, want LLM 可观测性 in the same pane.

### SigNoz ： OpenTelemetry-first full APM

- Apache 2.0. Handles general APM plus LLM via OpenTelemetry.
- Sweet spot: 统一 可观测性 across services and LLM calls.

### 胶水：OpenTelemetry + GenAI 语义约定

OpenTelemetry published GenAI semantic conventions in late 2025 (`gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`). Tools that consume OTel can interoperate. 这个生产化 模式 emerging:

1. Emit OTel with GenAI conventions from every LLM call.
2. 路由 to 网关 (Helicone / Portkey) for day-to-day.
3. Dual-ship to eval 平台 (Phoenix / Langfuse) for regressions.
4. Archive in data lake (Iceberg) for long-term analysis via Arize AX or DuckDB.

### 陷阱：在错误层做埋点

Instrumenting inside your agent framework (e.g., adding LangSmith 追踪) couples you to that framework. Instrumenting at the HTTP/OpenAI-SDK layer (via OpenLLMetry or your 网关) is portable.

### 采样：你不能保留一切

At >1M 请求/day, full-追踪 retention costs more than the LLM calls. Sample by rules: 100% errors, 100% high-成本, 5% success. Keep aggregates always; keep raw for the long 尾部.

### 你应该记住的数字

- Langfuse free cloud: 50K events/月.
- LangSmith: $39/user/月.
- Helicone free: 100K req/月.
- Arize AX claim: ~100x 更便宜 than monolithic at scale.
- OpenTelemetry GenAI conventions: 2025 shipping, 2026 widely adopted.

## 使用它

`code/main.py` simulates a 1M-追踪 day across retention strategies (100% ingest, sampling, sampling + errors). Reports storage 成本 and what's lost under each.

## 交付它

This lesson 产出 `outputs/skill-observability-stack.md`. Given stack, scale, budget, license posture, picks the tool(s).

## 练习

1. Your 团队 on LangChain wants OSS 自托管 可观测性. Pick Langfuse or Opik and 说明理由.
2. At 5M 追踪/day with Datadog quotes $150K/月, compute 盈亏平衡 for Arize AX.
3. Design an OpenTelemetry GenAI attribute set your org's guideline should mandate on every LLM call.
4. Argue whether Phoenix alone is sufficient for 生产化. When does it not suffice?
5. Helicone is 20ms proxy overhead. At P99 TTFT 300 ms, is that acceptable? What if SLA is 100 ms?

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| OpenLLMetry | "OTel for LLMs" | Open-source OpenTelemetry 埋点 for LLMs |
| GenAI conventions | "OTel attributes" | Standard OTel attribute names for LLM calls |
| LangSmith | "LangChain 可观测性" | Commercial 平台 bundled with LangChain ecosystem |
| Langfuse | "OSS LangSmith" | MIT OSS with 类似 功能 set |
| Phoenix | "Arize dev tool" | OpenTelemetry-native dev/eval 平台 |
| Arize AX | "scale 可观测性" | Commercial zero-copy Iceberg/Parquet 可观测性 |
| Helicone | "proxy 可观测性" | HTTP proxy collecting LLM telemetry + 网关 功能 |
| Opik | "Comet LLM" | Apache 2.0 OSS dev 平台 from Comet |
| Session replay | "追踪 rerun" | Replay a full agent session with tool calls |
| Eval | "offline test" | Running candidate 模型/提示词 over labeled dataset |

## 延伸阅读

- [SigNoz — Top LLM Observability Tools 2026](https://signoz.io/comparisons/llm-observability-tools/)
- [Langfuse — Arize AX Alternative analysis](https://langfuse.com/faq/all/best-phoenix-arize-alternatives)
- [PremAI — Setting Up Langfuse, LangSmith, Helicone, Phoenix](https://blog.premai.io/llm-observability-setting-up-langfuse-langsmith-helicone-phoenix/)
- [OpenTelemetry GenAI Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [Arize Phoenix docs](https://docs.arize.com/phoenix)
- [Helicone docs](https://docs.helicone.ai/)
