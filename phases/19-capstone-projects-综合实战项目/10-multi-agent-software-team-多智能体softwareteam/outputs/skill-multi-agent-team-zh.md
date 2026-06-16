---
name: multi-agent-team-zh
description: 建立一个由架构师、并行编码员、评审员和测试员组成的多代理软件团队；针对 SWE-bench Pro 进行测量并生成交接事后分析。
version: 1.0.0
phase: 19
lesson: 10
tags: [capstone, multi-agent, swe-bench, langgraph, a2a, worktree, roles]
---

给定 GitHub 问题 URL 和并行级别，部署一个多代理软件团队来生成可合并的 PR。评估 50 个 SWE-bench Pro 问题并发布切换失败直方图。
建设计划：
1. 任务板：类型化消息的文件支持（或 Redis）JSONL 存储。消息类型：plan_request、subtask、diff_ready、review_needed、review_feedback、approved、test_needed、test_passed、test_failed、replan_needed。
2. 架构师（Opus 4.7）：读取问题，编写计划，发出具有显式接口的子任务 DAG（涉及的文件、公共功能、测试影响）。
3. N 个编码员（Sonnet 4.7）：每个人声明一个子任务，生成一个新的 <span class="notranslate">`git worktree add`</span> + Daytona 沙箱，独立实现。
4、合并协调器：三路合并； LLM 介导的冲突解决仅针对文件级重叠。
5. Reviewer (GPT-5.4)：读​​取合并的 diff；无法批准其创作的差异；发出已批准或 review_feedback 路由到相关编码器。
6. 测试器（Gemini 2.5 Pro）：在干净的沙箱中运行测试套件；发出带有工件的 test_passed 或 test_failed 。
7. 切换记账：每条跨角色消息都成为一个带有负载大小和模型的Langfuse跨度。计算令牌放大=total_tokens / single_agent_baseline_tokens。
8. 注入明显的错误探针（运行的 10%）来测量审阅者错误批准率。
9. 运行 50 个 SWE-bench Pro 问题；发布 pass@1、挂钟与单代理基线、每个角色令牌细分、切换失败直方图。
评估标准：
|重量 |标准|测量|
|:-:|---|---|
| 25 | 25 SWE-bench Pro pass@1 | 50 个问题子集 pass@1 |
| 20 |并行加速 |挂钟与单代理基线 |
| 20 |审查质量 |注入错误探测器的错误批准率
| 20 |代币效率 |每个已解决问题的总代币与单一代理的比较 |
| 15 | 15协调工程|合并冲突解决、切换失败直方图 |
硬拒绝：
- 可以批准其撰写或提议的差异的审阅者。硬约束。
- 没有匹配的单代理基线运行的报告。多智能体必须赢得*每美元*，而不仅仅是通过@1。
- 任务板，其中的消息是自由格式的字符串，而不是键入的 A2A 消息。
- 合并协调器默默地丢弃冲突的差异，而不是路由回来重新计划。
拒绝规则：
- 拒绝在每个角色没有预算上限的情况下运行（代币+美元）。
- 拒绝打开测试人员未在干净沙箱中验证的 PR。
- 拒绝在单次运行中将编码器扩展至超过 8 个。协调开销在这之上占主导地位。
输出：包含任务板 + 角色工作人员的存储库、50 个问题的 SWE-bench Pro 运行日志、匹配的单代理基线运行、带有角色标记范围和每个角色令牌细分的 Langfuse 仪表板、注入错误探测报告以及事后剖析，其中命名了最常中断的三个切换以及减少每个切换的消息模式或提示更改。