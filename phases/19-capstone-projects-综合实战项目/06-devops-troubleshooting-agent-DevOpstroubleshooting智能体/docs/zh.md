# Capstone 06 — Kubernetes 的 DevOps 故障排查智能体

> AWS 的 DevOps Agent GA，Resolve AI 发布了 K8s playbooks，NeuBird 演示了 semantic monitoring，Metoro 把 AI SRE 绑定到 per-service SLO。生产形态已经稳定：alert webhook 触发，智能体读取 telemetry，遍历 K8s 对象图，排序 root-cause hypotheses，并发布带 approval buttons 的 Slack brief。默认只读。每个 remediation 都由人类 gate。本综合实战就是这个智能体，在 20 个 synthetic incidents 上评估，并在三个共享案例上与 AWS 的 Agent 对比。

**Type:** Capstone
**Languages:** Python（agent）、TypeScript（Slack integration）
**Prerequisites:** Phase 11（LLM engineering）、Phase 13（tools and MCP）、Phase 14（agents）、Phase 15（autonomous）、Phase 17（infrastructure）、Phase 18（safety）
**Phases exercised:** P11 · P13 · P14 · P15 · P17 · P18
**Time:** 30 小时

## 问题

2025-2026 年 SRE 叙事变成：“AI agents triage incidents, humans approve remediations.” AWS DevOps Agent、Resolve AI、NeuBird、Metoro、PagerDuty AIOps 都在生产中交付这种形态。智能体读取 Prometheus metrics、Loki logs、Tempo traces、kube-state-metrics，以及 K8s 对象知识图谱。它在五分钟内产出带 telemetry citations 的排序 root-cause hypothesis。没有通过 Slack 的明确 human approval，它绝不执行破坏性命令。

大多数困难工作在 scope 和 safety，而不是 reasoning。智能体需要默认只读的 RBAC surface、加固的 MCP tool server，以及每条被考虑命令与被执行命令的 audit logs。它需要知道自己何时超出能力范围并 escalate。它还必须足够便宜，避免 OOM-kill cascades 生成 5000 美元 agent 账单。

## 概念

智能体运行在知识图谱上。节点是 K8s objects（Pods、Deployments、Services、Nodes、HPAs、PVCs）加 telemetry sources（Prometheus series、Loki streams、Tempo traces）。边编码 ownership（Pod -> ReplicaSet -> Deployment）、scheduling（Pod -> Node）和 observation（Pod -> Prometheus series）。图通过 kube-state-metrics sync 保持新鲜，并在每个 alert 上重新采样。

当 alert 触发时，智能体从受影响对象开始 root-cause。它遍历边，拉取相关 telemetry slices（最近 15 分钟），并起草假设。假设按证据排序：有多少 telemetry citations 支持它，证据多新、多具体。Top-3 hypotheses 会发到 Slack，带 graph-path visualizations 和 remediation actions 的 approval buttons。

Remediation 被 gate。默认允许动作是只读的。破坏性动作（scaling down、rolling back、deleting Pods）需要 Slack approval；ArgoCD rollback hooks 需要一个 agent 从不持有的 auth token。Audit log 记录 agent 考虑过的每条命令，而不只是执行过的命令，所以 review 过程能捕捉 near-misses。

## 架构

```
PagerDuty / Alertmanager webhook
           |
           v
     FastAPI receiver
           |
           v
   LangGraph root-cause agent
           |
           +---- read-only MCP tools ----+
           |                             |
           v                             v
   K8s knowledge graph              telemetry slices
     (Neo4j / kuzu)              Prometheus, Loki, Tempo
   ownership + scheduling          last 15m, scoped
           |
           v
   hypothesis ranking (evidence weight)
           |
           v
   Slack brief + approval buttons
           |
           v (approved)
   ArgoCD rollback hook / PagerDuty escalate
           |
           v
   audit log: considered vs executed, every command
```

## 技术栈

- 可观测性来源：Prometheus、Loki、Tempo、kube-state-metrics
- 知识图谱：K8s objects + telemetry edges 的 Neo4j（托管）或 kuzu（嵌入式）
- Agent：LangGraph，带 per-tool allow-list，默认只读
- Tool transport：FastMCP over StreamableHTTP；破坏性工具放在 approval gate 后的独立 server
- Models：Claude Sonnet 4.7 用于 root-cause reasoning，Gemini 2.5 Flash 用于 log summarization
- Remediation：ArgoCD rollback webhook、PagerDuty escalate、Slack approval card
- Audit：append-only structured log（considered、executed、approved、outcome）
- Deployment：带独立窄 RBAC role 的 K8s deployment；独立 namespace

## 构建它

1. **图摄取。** 每 30 秒把 kube-state-metrics 同步到 Neo4j/kuzu。Nodes：Pod、Deployment、Node、Service、PVC、HPA。Edges：OWNED_BY、SCHEDULED_ON、EXPOSES、MOUNTS、SCALES。Telemetry overlay edges：OBSERVED_BY（一个 Pod 被一个 Prometheus series 观测）。

2. **Alert receiver。** FastAPI endpoint 接收 PagerDuty 或 Alertmanager webhooks。抽取受影响 object(s) 和 SLO breach。

3. **只读工具面。** 通过 FastMCP 包装 kubectl、Prometheus query、Loki logql、Tempo traceql。每个工具都有窄 RBAC verb（“get”、“list”、“describe”）。默认 server 中没有 “delete”、“exec”、“scale”。

4. **Root-cause agent。** LangGraph 有三个节点：`sample` 拉取最近 15 分钟 telemetry slice，`walk` 查询图上的邻近对象，`hypothesize` 起草带 telemetry citations 的排序 root-cause candidates。

5. **证据评分。** 每个 hypothesis 的 score = recency * specificity * graph-path length inverse * citation count。返回 top-3。

6. **Slack brief。** 发布 attachment，包含 hypothesis、graph-path visualization（server-side 渲染的 subgraph image），以及最多一个 remediation action 的 approval buttons。

7. **Remediation gate。** 破坏性工具（scale down、roll back、delete）位于第二个 MCP server 上，背后是 approval token。只有 Slack card 被人类批准后，agent 才能调用它们。

8. **Audit log。** Append-only JSONL：对每个 candidate command，记录它是否被 considered、是否 executed、谁 approved。每天发送到 S3。

9. **Synthetic incident suite。** 构建 20 个场景：OOMKill cascade、DNS flap、HPA thrash、PVC fill、noisy neighbor、faulty sidecar、bad ConfigMap rollout、certificate rotation、image-pull backoff 等。按 root-cause accuracy 和 time-to-hypothesis 给 agent 打分。

## 使用它

```
webhook: alert.pagerduty.com -> checkout-api SLO breach, error rate 14%
[graph]   affected: Deployment checkout-api (3 Pods, Node ip-10-2-3-4)
[walk]    neighbors: ReplicaSet checkout-api-abc, Service checkout-api,
           recent rollout 14m ago
[sample]  prometheus error_rate 14%, up-trend; loki 500s on /api/v2/pay
[hypo]    #1 bad rollout: latest image checkout-api:v2.41 fails /healthz
          citations: deploy.yaml (rev 42), prometheus errorRate, loki 500 stack
[slack]   [ROLL BACK to v2.40]  [ESCALATE]  [IGNORE]
          (approval required; agent does not roll back unilaterally)
```

## 交付它

`outputs/skill-devops-agent.md` 是交付物。给定一个 K8s cluster 和 alert source，智能体会产出排序 root-cause hypotheses 和 Slack-gated remediation flow。

| Weight | Criterion | How it is measured |
|:-:|---|---|
| 25 | 场景套件上的 RCA 准确率 | 20 个 synthetic incidents 中 ≥80% root cause 正确 |
| 20 | 安全 | audit log 中没有 Slack approval 时，破坏性动作 guard 从不放行 |
| 20 | Time-to-hypothesis | 从 alert 到 Slack brief 的 p50 低于 5 分钟 |
| 20 | 可解释性 | 每个 hypothesis 都有 graph paths 和 telemetry citations |
| 15 | 集成完整度 | PagerDuty、Slack、ArgoCD、Prometheus 端到端工作 |
| **100** | | |

## 练习

1. 在 AWS 的 DevOps Agent 演示过的三个相同 incidents 上运行你的 agent。发布 side-by-side。报告 agent 分歧在哪里。

2. 增加 “near-miss” audit，标记任何 agent 考虑过、如果没有 approval 就会是破坏性的命令。在一周内度量 near-miss rate。

3. 把 hypothesis model 从 Claude Sonnet 4.7 换成自托管 Llama 3.3 70B。度量 RCA accuracy delta 和 dollar per incident。

4. 构建 causal filter：区分相关 telemetry spikes 和真正 root cause。用 20-scenario labels 训练一个小 classifier。

5. 增加 rollback dry-run：在同 manifest 的 staging cluster 上执行 ArgoCD rollback。Slack approval button 之前，在 live cluster 中验证 rollback plan。

## 关键术语

| Term | What people say | What it actually means |
|------|-----------------|------------------------|
| K8s knowledge graph | “Cluster graph” | Nodes = K8s objects + telemetry series；edges = ownership、scheduling、observation |
| Read-only-by-default | “Scoped RBAC” | Agent 的 service account 只有 get/list/describe verbs；破坏性 verbs 位于 approval 后的独立 server |
| Audit log | “Considered vs executed” | 每个 candidate command 的 append-only record，记录是否运行、谁批准 |
| Hypothesis ranking | “Evidence score” | Recency × specificity × graph-path length inverse × citation count |
| Slack approval card | “HITL gate” | 带 remediation buttons 的交互式 Slack message；人类点击前 agent 不能继续 |
| Telemetry citation | “Evidence pointer” | 支持某个主张的 Prometheus query、Loki selector 或 Tempo trace URL |
| MTTR | “Time to resolution” | 从 alert fire 到 SLO recovery 的墙钟时间 |

## 延伸阅读

- [AWS DevOps Agent GA](https://aws.amazon.com/blogs/aws/aws-devops-agent-helps-you-accelerate-incident-response-and-improve-system-reliability-preview/) — 2026 年标准参考
- [Resolve AI K8s troubleshooting](https://resolve.ai/blog/kubernetes-troubleshooting-in-resolve-ai) — 竞品参考
- [NeuBird semantic monitoring](https://www.neubird.ai) — semantic-graph approach
- [Metoro AI SRE](https://metoro.io) — SLO-first 生产 framing
- [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) — cluster-state source
- [LangGraph](https://langchain-ai.github.io/langgraph/) — 参考 agent orchestrator
- [FastMCP](https://github.com/jlowin/fastmcp) — Python MCP server framework
- [ArgoCD rollback](https://argo-cd.readthedocs.io/en/stable/user-guide/commands/argocd_app_rollback/) — gated remediation target
