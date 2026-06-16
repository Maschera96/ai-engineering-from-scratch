# 大模型生产化的混沌工程

> 混沌工程 for LLMs is its own discipline in 2026. Prerequisites before running experiments in 生产化: defined SLI/SLO, 追踪+指标+log 可观测性, automated 回滚, runbooks, on-call. Architecture has four planes: control (experiment 调度器), target (services, infra, data stores), safety (guards + abort + 流量 filters), 可观测性 (指标 + 追踪 + 日志), feedback (into SLO adjustments). 护栏 are mandatory: burn-rate alerts pause experiments if daily error-budget burn > 2x expected; suppression windows + 追踪-ID correlation dedupe alert noise. Cadence: weekly small 金丝雀 + SLO review; monthly game day + postmortem; quarterly cross-团队 resilience audit + dependency mapping. LLM-具体 experiments: 内存 overload, network failures, 提供商 故障, malformed 提示词, KV 缓存 eviction storms. Tooling: Harness 混沌工程 (LLM-derived recommendations, blast-radius downscaling, MCP tool integration); LitmusChaos (CNCF); Chaos Mesh (CNCF Kubernetes-native).

**Type:** Learn
**Languages:** Python（标准库， 玩具 chaos experiment runner)
**Prerequisites:** Phase 17 · 23 (SRE for AI), Phase 17 · 13 (Observability)
**Time:** ~60 分钟

## 学习目标

- Name the five 混沌工程 prerequisites (SLI/SLO, 可观测性, 回滚, runbooks, on-call) 和 解释 why skipping any breaks the practice.
- Diagram the four planes (control, target, safety, 可观测性) and the feedback loop into SLO.
- Enumerate five LLM-具体 experiments (内存 overload, network fail, 提供商 outage, malformed 提示词, KV eviction storm).
- Pick a tool ： Harness, LitmusChaos, Chaos Mesh ： given stack.

## 问题

Chaos testing in traditional stacks is established. LLM stacks add new failure modes. A 4K-词元 提示词 with a poison character stalls the tokenizer for 12 seconds. An upstream 提供商 429s; your 网关 retries; your service OOMs on retry-amplified concurrency. A KV 缓存 eviction storm under burst load causes re-prefill cascades that saturate compute.

None of these show up in unit tests. 混沌工程 is how you 发现 them before users do.

## 概念

### 先修要求

Don't run chaos in 生产化 without:

1. **SLI/SLO** ： defined service-level indicators and objectives.
2. **可观测性** ： 追踪, 指标, 日志, wired to dashboards.
3. **Automated 回滚** ： Phase 17 · 20 策略-flag 回滚.
4. **Runbooks** ： structured, Phase 17 · 23.
5. **On-call** ： someone to respond.

Missing any means chaos becomes real 事故.

### Four planes + feedback

**Control plane** ： experiment 调度器 (Litmus workflow, Chaos Mesh schedule, Harness UI).

**Target plane** ： services, pods, nodes, load balancers, data stores.

**Safety plane** ： kill switch, suppression windows, blast-radius limits, error-budget gates.

**可观测性 plane** ： normal 指标 + 追踪-ID correlation to distinguish chaos-induced from natural failures.

**Feedback loop** ： findings feed back into SLO adjustment, 运行手册 updates, code fixes.

### 护栏是必需项

- **Burn-rate alert**: pause experiment if daily error-budget burn exceeds 2x expected.
- **Suppression windows**: silence non-experiment alerts in the blast radius during experiment.
- **追踪-ID correlation**: all experiment-induced errors carry a tag so on-call can dedupe.

### Five LLM-具体 experiments

1. **内存 overload** ： force a KV 缓存 preemption storm by sending 长上下文 请求 with high concurrency. Observe: does the service gracefully shed or crash?

2. **Network failure** ： cut connectivity between inference 网关 和 提供商. Observe: does fallback kick in within SLA? (Phase 17 · 19)

3. **提供商 outage simulation** ： 100% 429 from OpenAI. Observe: does 路由 故障切换 to Anthropic? (Phase 17 · 16, 19)

4. **Malformed 提示词** ： inject tokenizer-stalling payload (e.g., deeply nested unicode, huge UTF-8 codepoint). Observe: does a single 请求 lock up a worker?

5. **KV eviction storm** ： force eviction by saturating vLLM block budget. Observe: does LMCache recover or does service degrade?

### 节奏

- **Weekly** ： small 金丝雀 experiments in staging, maybe 5% prod.
- **Monthly** ： scheduled game day on 一个具体 scenario; cross-团队 attendance; postmortem.
- **Quarterly** ： cross-团队 resilience audit; dependency map update.

### 工具链

- **Harness 混沌工程** ： commercial; AI-derived experiment recommendations; blast-radius downscaling; MCP tool integration.
- **LitmusChaos** ： CNCF graduated; Kubernetes workflow-based.
- **Chaos Mesh** ： CNCF sandbox; Kubernetes-native CRD style.
- **Gremlin** ： commercial; broad support.
- **AWS FIS** / **Azure Chaos Studio** ： managed cloud offerings.

### 从小处开始

First experiment: pod-kill one decode replica under steady 流量. Observe rerouting and recovery. If this works and looks safe, graduate to network chaos.

First LLM-具体 experiment: inject one 提供商 429 for 5 minutes. Observe fallback. Most 团队 发现 their fallback wasn't fully tested.

### 你应该记住的数字

- Four planes: control, target, safety, 可观测性.
- Burn-rate pause: 2x expected daily budget burn.
- Cadence: weekly 金丝雀, monthly game day, quarterly audit.
- Five LLM experiments: 内存, network, 提供商, malformed 提示词, KV storm.

## 使用它

`code/main.py` simulates three chaos experiments with safety plane gates. Reports which experiments would trip the burn-rate abort.

## 交付它

This lesson 产出 `outputs/skill-chaos-plan.md`. Given stack and maturity, picks first three experiments and the tooling.

## 练习

1. Run `code/main.py`. Which experiment trips the burn-rate gate and why?
2. Design the first five chaos experiments for a vLLM-based RAG service. Include success criteria.
3. Your burn-rate alert paused an experiment. How do you determine root cause ： chaos or natural?
4. Argue whether chaos should run in 生产化 or only staging. When is 生产化 the right answer?
5. Name three LLM-具体 failure modes that generic network-chaos cannot reproduce.

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| SLI / SLO | "service targets" | Indicator + objective; 必需 prerequisite |
| Blast radius | "scope" | Set of services / users affected by experiment |
| Burn-rate alert | "budget gate" | Fires when error-budget burn rate > 2x expected |
| Game day | "monthly drill" | Scheduled cross-团队 chaos exercise |
| LitmusChaos | "CNCF workflow" | Graduated CNCF Kubernetes chaos tool |
| Chaos Mesh | "CNCF CRD" | CNCF sandbox Kubernetes-native chaos |
| Harness CE | "commercial AI-assisted" | Harness chaos with AI recommendations |
| Malformed 提示词 | "tokenizer bomb" | 输入 that stalls tokenization |
| KV eviction storm | "preemption cascade" | Mass eviction triggering re-prefills |

## 延伸阅读

- [DevSecOps School — Chaos Engineering 2026 Guide](https://devsecopsschool.com/blog/chaos-engineering/)
- [Ankush Sharma — Observability for LLMs (book)](https://www.amazon.com/Observability-Large-Language-Models-Engineering-ebook/dp/B0DJSR65TR)
- [LitmusChaos (CNCF)](https://litmuschaos.io/)
- [Chaos Mesh (CNCF)](https://chaos-mesh.org/)
- [Harness Chaos Engineering](https://www.harness.io/products/chaos-engineering)
- [AWS FIS](https://aws.amazon.com/fis/)
