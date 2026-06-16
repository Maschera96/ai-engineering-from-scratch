---
name: terminal-coding-agent-zh
description: 使用有限成本、沙盒工具和完整的 2026 钩子表面，针对 SWE-bench Pro 构建和评估终端本机编码代理。
version: 1.0.0
phase: 19
lesson: 01
tags: [capstone, coding-agent, claude-code, swe-bench, mcp, hooks, sandbox]
---

给定目标存储库和自然语言任务，构建一个在沙箱中规划、执行并打开拉取请求的工具。在包含 30 个任务的 SWE-bench Pro 子集中匹配或超越 mini-swe-agent 基线，同时将每任务预算控制在 5 美元以下。
建设计划：
1. 建立一个 Bun + Ink TUI 工具，其中包含计划窗格、工具调用流和实时 token/dollar 预算。
2. 通过模型上下文协议 StreamableHTTP 定义六个工具（read_file、edit_file、ripgrep、tree_sitter_symbols、run_shell、git）。每次调用最多返回 4k 令牌。
3. 在新的 `git worktree add` 分支上的 E2B 或 Daytona 沙箱内运行每个工具调用。切勿触摸主机文件系统。
4. 连接所有八个 2026 挂钩事件：SessionStart、SessionEnd、PreToolUse、PostToolUse、UserPromptSubmit、Notification、Stop、PreCompact。提供至少四个用户编写的钩子（破坏性命令防护、令牌记账、OTel 跨度发射器、跟踪包编写器）。
5. 执行三项预算：50 轮、20 万代币、5 美元。 PreCompact 在 150k 时触发并总结了较旧的转弯。
6. 将具有 GenAI 语义约定的 OpenTelemetry 跨度发送到自托管 Langfuse。
7. 成功后，推送分支并打开一个 PR，其中包含计划和跟踪包。
8. 在 30 个问题的 SWE-bench Pro Python 子集上针对 mini-swe-agent 进行评估，并记录每个任务的 pass@1、回合、代币和美元。
评估标准：
|重量 |标准|测量|
|:-:|---|---|
| 25 | 25 SWE-bench Pro pass@1 |匹配的 30 任务子集与 mini-swe-agent 基线 |
| 20 |架构清晰度 | Plan/act/observe 分离、钩子表面、工具模式可读性 |
| 20 |安全|沙盒越狱红队+破坏性指挥警卫审核|
| 20 |可观察性| 100% 的工具调用跨越，每回合代币记账 |
| 15 | 15开发人员用户体验 | 2 秒以下冷启动、崩溃恢复、Ctrl-C 取消语义 |
硬拒绝：
- 在主机文件系统上而不是在沙箱内使用 git 的工具。
- 任何可以在工作树外部写入或卷曲外部 URL 且无需显式允许列表挂钩的代理。
- 报告的评估数字没有针对相同的 30 个问题运行匹配的基线。
- “通过率”声明取决于重试之间的 `git reset --hard`； SWE-bench Pro 是 pass@1。
拒绝规则：
- 拒绝在任何配置下直接推送到 main。仅限公关分支机构。
- 拒绝禁用破坏性命令防护。这是该准则的硬性要求。
- 拒绝在没有预算上限的情况下运行。开放式运行会污染评估比较。
输出：包含该工具的存储库、一个固定的 30 任务 SWE-bench Pro 评估工具（具有匹配的 mini-swe-agent 基线运行）、至少 5 次完整运行的 OpenTelemetry 跟踪存档，以及一个书面命名工具解决了基线无法解决的任务，反之亦然。最后一节介绍了您观察到的前三种故障模式以及修复每个故障模式的钩子更改。