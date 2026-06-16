# 面向 AI 的 SRE：多智能体事故响应、运行手册、预测性检测

> AI SRE uses LLMs grounded in 基础设施 data (日志, runbooks, service topology) via RAG to automate investigation, documentation, and coordination phases. The 2026 architecture 模式 is multi-agent orchestration ： specialized agents (日志, 指标, runbooks) coordinated by a supervisor; AI proposes hypotheses and queries, humans approve judgment calls. Datadog Bits AI and Azure SRE Agent ship this as managed products. Runbooks are evolving: NeuBird Hawkeye uses adversarial 评估 (two 模型 analyze the same 事故; agreement = confidence, disagreement = uncertainty); operational 内存 persists across 团队 changes. Auto-remediation stays cautious: AI suggests, humans approve. Fully autonomous action is narrow (restart pod, 回滚 具体 deploy) with tight 护栏 ： anyone selling "set it and forget it" is overselling. Emerging frontier: pre-事故 prediction. MIT research reports an LLM trained on historical 日志 + GPU temps + API error patterns predicted 89% of 故障 10-15 min early. Projection: 95% of 企业级 LLMs have automated 故障切换 by end-2026.

**Type:** Learn
**Languages:** Python（标准库， 玩具 multi-agent incident triage模拟器)
**Prerequisites:** Phase 17 · 13 (Observability), Phase 17 · 24 (Chaos Engineering)
**Time:** ~60 分钟

## 学习目标

- Diagram the multi-agent AI SRE architecture: supervisor + specialized agents (日志, 指标, runbooks) + human approval gate.
- 解释 why auto-remediation is narrow (restart pod, revert deploy) rather than broad (re-architect service).
- Name the adversarial 评估 模式 (NeuBird Hawkeye): two 模型 agree = confidence; disagree = escalate.
- Cite the MIT 89% early-detection result and the operational constraint: predictions without actuation are just dashboards.

## 问题

An on-call engineer gets paged at 3 a.m. "High error rate in checkout." They check Datadog, Loki, three runbooks, the deploy log. 30 minutes later they realize the root cause is a vLLM OOM from a KV 缓存 spike. They restart the pod; error clears.

In 2026 the first 20 minutes of that investigation are automatable. Grouping 日志 by service, correlating to recent deploys, matching against runbooks ： all are RAG + tool-use. A supervised agent can do first-pass triage and present a hypothesis before the human opens Datadog.

Fully autonomous remediation is a different problem. Restart pod: safe. Scale GPU pool: safe if 策略 allows. Re-architect the service: absolutely not. The discipline is drawing the narrow line.

## 概念

### 多智能体架构

```
          Incident
             │
             ▼
        Supervisor
        /    |    \
       ▼     ▼     ▼
  Log agent  Metric agent  Runbook agent
       │     │     │
       └─────┴─────┘
             │
             ▼
        Hypothesis + evidence
             │
             ▼
        Human approval
             │
             ▼
        Action (narrow set)
```

Supervisor breaks 这个事故 into sub-queries. Specialized agents have tool access (log search, PromQL, doc retrieval). Supervisor synthesizes, presents hypothesis + evidence to human. Human approves or redirects.

### 自动修复范围

**Safe (narrow)**: restart pod, revert 具体 deploy, scale pool within pre-approved bounds, enable pre-approved 功能 flag.

**Not safe (broad)**: change service topology, modify resource limits, deploy new code, change IAM, alter databases.

Anyone selling "set it and forget it" is overselling. The safe set grows as AI SRE matures, but the boundary is real.

### Adversarial 评估 (NeuBird Hawkeye)

Two 模型 independently analyze the same 事故. If they agree on root cause, confidence is high. If they disagree, escalate to human with both hypotheses visible. 简单 模式, effective filter against hallucinated root causes.

### 运维记忆

团队 turnover is the silent kill of traditional SRE ： tribal knowledge leaves. AI SRE stores runbooks + post-mortems in a vector DB; agents retrieve on every new 事故. When new engineers join, the AI has full history.

### Pre-事故 prediction

MIT 2025 research: LLM trained on historical 日志, GPU temperatures, API error patterns predicted 89% of 故障 10-15 minutes before they happened on the test set.

Reality check: predictions without actuation are dashboards. The operational question is "when we predict, what do we do?" Pre-emptive drain? Pager? Auto-scale? The answer is 策略-具体.

### 2026 年产品

- **Datadog Bits AI** ： managed SRE copilot inside Datadog.
- **Azure SRE Agent** ： Azure-native.
- **NeuBird Hawkeye** ： adversarial eval + operational 内存.
- **PagerDuty AIOps** ： triage + deduplication.
- **事故.io Autopilot** ： 事故 commander + coordination.

### 运行手册即代码

Runbooks evolve from Confluence pages to versioned markdown with structured sections (symptom, hypothesis, verify, act). Structured runbooks feed better RAG retrieval. Start any AI-SRE rollout by turning unstructured runbooks into structured.

### 你应该记住的数字

- MIT early-detection: 89% of 故障, 10-15 min lead time.
- Multi-agent triage: supervisor + (日志, 指标, runbooks) + human.
- Safe auto-remediation set: restart pod, revert deploy, scale within bounds.
- Adversarial eval: two 模型 independent; agreement = confidence.

## 使用它

`code/main.py` simulates a multi-agent triage: log agent finds error, 指标 agent finds CPU spike, 运行手册 agent matches to known issue. Supervisor ranks hypotheses.

## 交付它

This lesson 产出 `outputs/skill-ai-sre-plan.md`. Given current on-call, 事故 volume, 团队 maturity, designs an AI SRE rollout.

## 练习

1. Run `code/main.py`. What if the log and 指标 agents disagree? How does the supervisor resolve?
2. Define three "safe" auto-remediation actions for your service. 说明理由 each.
3. Write a structured 运行手册 template: sections, 必需 fields, verification commands.
4. Predictive detection fires at 12 min lead. What's your 策略 ： pager, pre-drain, or both?
5. Argue whether a 3-person 团队 should 采用 AI SRE in 2026 or wait. Consider maturity, volume, 风险.

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| AI SRE | "agent for on-call" | LLM-backed 事故 investigation + coordination |
| Supervisor agent | "the orchestrator" | Top-level agent breaking incidents into sub-queries |
| Specialized agent | "domain agent" | Sub-agent with tool access (日志, 指标, runbooks) |
| Auto-remediation | "AI fixes it" | Narrow pre-approved action; NOT broad re-architecture |
| Operational 内存 | "vector runbooks" | Post-mortems + runbooks in vector DB for RAG |
| Adversarial eval | "two-模型 check" | Independent analyses; agreement = confidence |
| NeuBird Hawkeye | "the adversarial one" | 产品 with adversarial-eval + 内存 模式 |
| Bits AI | "Datadog's SRE agent" | Datadog-managed AI SRE |
| Pre-事故 prediction | "early detection" | 10-15 min lead time on outage prediction |

## 延伸阅读

- [incident.io — AI SRE Complete Guide 2026](https://incident.io/blog/what-is-ai-sre-complete-guide-2026)
- [InfoQ — Human-Centred AI for SRE](https://www.infoq.com/news/2026/01/opsworker-ai-sre/)
- [DZone — AI in SRE 2026](https://dzone.com/articles/ai-in-sre-whats-actually-coming-in-2026)
- [Datadog Bits AI](https://www.datadoghq.com/product/bits-ai/)
- [NeuBird Hawkeye](https://www.neubird.ai/)
- [awesome-ai-sre](https://github.com/agamm/awesome-ai-sre)
