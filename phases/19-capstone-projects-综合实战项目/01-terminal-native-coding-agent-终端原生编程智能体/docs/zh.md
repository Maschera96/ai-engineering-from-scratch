# Capstone 01 — 终端本机编码代理
> 到 2026 年，编码剂的形状已经确定。 TUI 工具、状态计划、沙盒工具表面、计划、行动、观察、恢复的循环。从 50 英尺处看，Claude Code、Cursor 3 和 OpenCode 看起来都一样。此顶点要求您构建端到端 - CLI 输入，拉出请求 - 并根据 SWE-bench Pro 上的 mini-swe-agent 和 Live-SWE-agent 对其进行衡量。您将了解为什么困难的部分不是模型调用，而是工具循环、沙箱和 50 回合运行的成本上限。
**类型：** 顶点
**语言：** TypeScript / Bun（线束）、Python（评估脚本）
**先决条件：** 第 11 阶段（LLM 工程）、第 13 阶段（工具和协议）、第 14 阶段（代理）、第 15 阶段（自治系统）、第 17 阶段（基础设施）
**练习阶段：** P0 · P5 · P7 · P10 · P11 · P13 · P14 · P15 · P17 · P18
**时间：** 35小时
＃＃ 问题
编码代理在 2026 年成为占主导地位的 AI 应用类别。Claude Code (Anthropic)、带有 Composer 2 和 Agent Tabs (Cursor) 的 Cursor 3、Amp (Sourcegraph)、OpenCode (112k star)、Factory Droids 和 Google Jules 都提供了相同架构的变体：终端线束、许可工具表面、沙箱以及围绕前沿构建的计划-行动-观察循环模型。边界很窄——Live-SWE-agent 在 SWE-bench 上通过 Opus 4.5 验证达到了 79.2%——但工程技术很宽。大多数失效模式不是模型错误。它们是工具循环不稳定、上下文中毒、令牌成本失控和破坏性文件系统操作。
你无法从外部推断这些代理。您必须构建一个，在 ripgrep 返回 8MB 匹配项时观察第 47 回合循环崩溃，并重建截断层。这就是这个顶点的要点。
＃＃ 概念
线束有四个表面。 **Plan** 维护一个 TodoWrite 风格的状态对象，模型每轮都会重写该对象。 **Act** 调度工具调用（读取、编辑、运行、搜索、git）。 **Observe** 捕获 stdout / stderr / 退出代码、截断并反馈摘要。 **恢复** 处理工具错误，而不会破坏上下文窗口或永远循环。 2026 年的形状又增加了一项：**挂钩**。 `PreToolUse`、`PostToolUse`、`SessionStart`、`SessionEnd`、`UserPromptSubmit`、`Notification`、`Stop` 和 `PreCompact` — 可配置的扩展点，操作员可在其中注入策略、遥测和护栏。
沙盒是E2B或Daytona。每个任务都在一个新的开发容器中运行，并安装了读写的 git 工作树。该工具从不接触主机文件系统。工作树会根据成功或失败而被拆除。成本控制在三层实施：每轮代币上限、每会话美元预算和硬轮限制（通常为 50）。可观察性层是 OpenTelemetry 跨越 GenAI 语义约定，运送到自托管的 Langfuse。
＃＃ 建筑学
```
  user CLI  ->  harness (Bun + Ink TUI)
                  |
                  v
           plan / act / observe loop  <--->  Claude Sonnet 4.7 / GPT-5.4-Codex / Gemini 3 Pro
                  |                          (via OpenRouter, model-agnostic)
                  v
           tool dispatcher (MCP StreamableHTTP client)
                  |
     +------------+------------+----------+
     v            v            v          v
  read/edit    ripgrep     tree-sitter   git/run
     |            |            |          |
     +------------+------------+----------+
                  |
                  v
           E2B / Daytona sandbox  (worktree isolated)
                  |
                  v
           hooks: Pre/Post, Session, Prompt, Compact
                  |
                  v
           OpenTelemetry -> Langfuse (spans, tokens, $)
                  |
                  v
           PR via GitHub app
```

＃＃ 堆
- Harness 运行时：Bun 1.2 + Ink 5（在终端中反应）
- 模型访问：OpenRouter统一API与Claude Sonnet 4.7、GPT-5.4-Codex、Gemini 3 Pro、Opus 4.5（适用于最困难的任务）
- 工具传输：模型上下文协议 StreamableHTTP（MCP 2026 修订版）
- 沙箱：E2B 沙箱（JS SDK）或 Daytona devcontainers
- 代码搜索：ripgrep 子进程，17 种语言的树托管解析器（预编译）
- 隔离：每个任务 `git worktree add`，成功/失败时清理
- 评估工具：SWE-bench Pro（经过验证的子集）+ Terminal-Bench 2.0 + 您自己的 30 任务保留
- 可观察性：带有 `gen_ai.*` semconv 的 OpenTelemetry SDK → 自托管 Langfuse
- PR 发布：具有细粒度令牌的 GitHub 应用程序，范围仅限于目标存储库
## 构建它
1. **TUI 和命令循环。** 使用 Ink 搭建 Bun 项目。接受 `agent run <repo> "<task>"`。打印分割视图：计划窗格（顶部）、工具调用流（中间）、令牌预算（底部）。在 Ctrl-C 上添加取消，在退出前触发 `SessionEnd` 挂钩。
2. **计划状态。** 定义类型化的 TodoWrite 模式（带有注释的待处理/进行中/已完成项目）。模型每轮都会将完整状态重写为工具调用 - 不要让它增量变异。坚持 `.agent/state.json` 计划，以便崩溃可以恢复。
3. **工具面。** 定义六个工具：`read_file`、`edit_file`（带 diff 预览）、`ripgrep`、`tree_sitter_symbols`、`run_shell`（带超时）、`git`（状态/diff/commit/push）。通过 MCP StreamableHTTP 公开，因此该工具与传输无关。每个工具都会返回截断的输出（每次调用上限为 4k 个令牌）。
4. **沙箱包装。** 每个任务都会生成一个 E2B 沙箱。 `git worktree add -b agent/$TASK_ID` 是一个新分支。所有工具调用都在沙箱内执行。主机文件系统无法访问。
5. **钩子。** 实现所有八种 2026 钩子类型。连接至少四个用户编写的钩子：(a) `PreToolUse` 破坏性命令防护，阻止工作树外部的 `rm -rf`，(b) `PostToolUse` 令牌记账，(c) `SessionStart` 预算初始化，(d) `Stop` 写入最终跟踪包。
6. **Eval 循环。** 克隆 SWE-bench Pro Python 的 30 个问题子集。将你的安全带对着每一个。与 pass@1、turns-per-task 和 $-per-task 上的 mini-swe-agent（最小基线）进行比较。将结果写入 `eval/results.jsonl`。
7. **成本控制。** 硬性截止：50 轮，200k 上下文，每个任务 5 美元。 `PreCompact` 钩子将旧的转弯总结为 150k 标记处的先前状态块，为新观察释放空间而不会丢失计划。
8. **PR 发布。** 成功后，最后一步是 `git push` + GitHub API 调用，该调用将打开 PR，其中包含计划和正文中的差异摘要。
## 使用它
```
$ agent run ./my-repo "Fix the race condition in worker.rs"
[plan]  1 locate worker.rs and enumerate mutex uses
        2 identify shared state under contention
        3 propose fix, verify tests
[tool]  ripgrep mutex.*lock -t rust           (44 matches, truncated)
[tool]  read_file src/worker.rs 120..180
[tool]  edit_file src/worker.rs (+8 -3)
[tool]  run_shell cargo test worker::          (passed)
[plan]  1 done · 2 done · 3 done
[done]  PR opened: #482   turns=9   tokens=38k   cost=$0.41
```

## 发货
可交付的技能位于 `outputs/skill-terminal-coding-agent.md` 中。给定存储库路径和任务描述，它会在沙箱中运行完整的计划-行动-观察循环，并返回 PR URL 和跟踪包。此顶点的标题：
|重量 |标准|如何测量 |
|:-:|---|---|
| 25 | 25 SWE-bench Pro pass@1 与基线 |您的harness与mini-swe-agent在30个匹配的Python任务上的对比|
| 20 |架构清晰度 | Plan/act/observe 分离、钩子表面、工具模式 — 根据 Live-SWE-agent 布局进行审查 |
| 20 |安全|沙箱逃逸测试、权限提示、破坏性命令守卫通过红队 |
| 20 |可观察性|跟踪完整性（跨越 100% 的工具调用）、每轮令牌记账 |
| 15 | 15开发人员用户体验 |冷启动 < 2 秒，崩溃恢复恢复计划，Ctrl-C 干净地取消中间工具 |
| **100** | | |
## 练习
1. 将支持模型从 Claude Sonnet 4.7 交换为 vLLM 上服务的 Qwen3-Coder-30B。比较 pass@1 和 $-per-task。报告开放模型表现不佳的地方。
2. 添加一个 `reviewer` 子代理，该子代理在 PR 发布之前读取差异并可以请求修订循环。衡量假阳性评论是否会将 SWE 基准通过率降低到单代理基线以下（提示：通常是）。
3. 对沙箱进行压力测试：编写一个尝试 `curl` 外部 URL 的任务和一个在工作树外部写入的任务。确认两者都被 PreToolUse 挂钩阻止。记录尝试。
4. 使用较小的模型（Haiku 4.5）实现 `PreCompact` 汇总。测量 3 倍压缩时计划保真度损失了多少。
5. 将 MCP StreamableHTTP 传输替换为 stdio。基准冷启动和每次调用延迟。选择仅供本地使用的获胜者。
## 关键术语
|术语 |人们怎么说|它实际上意味着什么 ||------|-----------------|------------------------|
|线束| “代理循环”|围绕模型的代码，用于调度工具、维护计划状态和执行预算 |
|钩| “代理事件侦听器” |用户编写的脚本通过线束在八个生命周期事件之一上运行 |
|工作树 | “Git 沙箱” |在单独的路径上链接的 git checkout；一次性，无需接触主克隆|
| TodoWrite | “计划状态” |模型每回合都会重写的 pending/in-progress/done 项的键入列表 |
| StreamableHTTP | “MCP 传输”| 2026 MCP 修订版：具有双向流的长期 HTTP 连接；取代上交所 |
|代币上限 | “背景预算”|输入+输出代币的每回合或每会话上限；触发压缩或终止 |
|通过@1 | “单次尝试通过率” | SWE 基准任务的一部分在第一次运行时就得到解决，无需重试或查看测试集 |
## 进一步阅读
- [Claude 代码文档](https://docs.anthropic.com/en/docs/claude-code) — 来自 Anthropic 的参考工具
- [Cursor 3 变更日志](https://cursor.com/changelog) — Agent 选项卡和 Composer 2 产品说明
- [mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent) — SWE 工作台线束比较的最小基线
- [Live-SWE-agent](https://github.com/OpenAutoCoder/live-swe-agent) — 79.2% SWE-bench 通过 Opus 4.5 进行验证
- [OpenCode](https://opencode.ai) — 开放线束，112k 星
- [SWE-bench Pro 排行榜](https://www.swebench.com) — 此顶点目标的评估
- [模型上下文协议 2026 路线图](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/) — StreamableHTTP，功能元数据
- [OpenTelemetry GenAI 语义约定](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — 工具调用和令牌使用的跨架构