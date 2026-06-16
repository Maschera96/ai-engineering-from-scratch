---
name: devops-agent-zh
description: 构建一个 Kubernetes 故障排除代理，该代理会遍历集群知识图、对根本原因进行排名并通过 Slack 控制每个补救措施。
version: 1.0.0
phase: 19
lesson: 06
tags: [capstone, devops, sre, kubernetes, langgraph, fastmcp, aiops]
---

给定一个 K8s 集群和一个警报源（PagerDuty 或 Alertmanager），构建一个代理，该代理可以在五分钟内生成排序的根本原因假设，并通过 Slack 批准卡控制每个补救措施。
建设计划：
1. 每 30 秒将 kube-state-metrics 摄取到 Neo4j 或 kuzu 中。构建 Pod、部署、服务、节点、PVC、HPA 以及到 Prometheus、Loki 和 Tempo 源的遥测覆盖边缘的图表。
2. 为 PagerDuty 和 Alertmanager 建立 FastAPI Webhook 接收器。
3. 通过带有 StreamableHTTP 传输的 FastMCP 公开只读工具：kubectl get/describe、promql、logql、traceql。
4. 构建具有三个节点的 LangGraph 根本原因代理：<span class="notranslate">`sample`</span>（拉动 15m 遥测）、<span class="notranslate">`walk`</span>（遍历图邻居），<span class="notranslate">`hypothesize`</span>（按新近度 × 特异性 × 引用计数对候选者进行排名）。
5. 将带有图形路径可视化的排名前 3 的假设发布到带有批准按钮的 Slack。
6. 将破坏性工具（扩展、回滚、删除）放在单独的 FastMCP 服务器上，位于代理仅在 Slack 签核后才获得的批准令牌后面。
7. 维护仅附加审核日志：每个*考虑的*命令，是否批准，是否执行，谁批准。
8. 构建 20 个综合事件场景（OOMKill、DNS 抖动、HPA 颠簸、PVC 填充、嘈杂的邻居、故障 sidecar、ConfigMap 错误部署、证书轮换、图像拉取退避、探测失败等 10 个）。对代理的 RCA 准确性和假设时间进行评分。
评估标准：
|重量 |标准|测量|
|:-:|---|---|
| 25 | 25情景套件的 RCA 精度 | 20 起综合事件中至少 80% 的根本原因是正确的 |
| 20 |安全|未经 Slack 审核日志批准，破坏性动作守卫绝不会开火 |
| 20 |假设时间 | p50 从警报到 Slack 简报不到 5 分钟 |
| 20 |可解释性|每个假设都有图形路径和遥测引用 |
| 15 | 15集成完整性 | PagerDuty、Slack、ArgoCD、Prometheus 端到端工作 |
硬拒绝：
- 具有单个 MCP 服务器的代理，该服务器混合了只读和破坏性工具。
- 任何没有遥测引用的 RCA。必须拒绝未引用的假设。
- 仅记录执行情况的审核日志。他们必须记录每条考虑的命令。
- 无需针对带有种子的 20 个场景套件运行代理即可声称准确。
拒绝规则：
- 未经呼叫人员的 Slack 批准，拒绝补救。即使假设是显而易见的。
- 拒绝通过只读 <span 公开 <span class="notranslate">`kubectl exec`</span>、<span class="notranslate">`kubectl port-forward`</span> 或任何交互式工具类=“notranslate”>MCP</span>。这些实际上是破坏性的。
- 拒绝在没有每个部署批准卡的情况下跨多个部署批量应用修复。
输出：一个存储库，其中包含 FastAPI 接收器、LangGraph 代理、只读且具有破坏性的 MCP 服务器、Slack 集成、20 个场景测试套件、在三个共享事件上与 AWS DevOps 代理进行并排比较，以及在为期一周的观察窗口中记录未遂命令（代理“考虑”但未执行的命令）。