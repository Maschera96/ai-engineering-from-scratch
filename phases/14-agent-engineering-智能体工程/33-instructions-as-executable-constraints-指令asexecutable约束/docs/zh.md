# Agent Instructions as Executable Constraints

> 以散文形式编写的指令只是愿望，以约束形式编写的指令才是测试。工作台（workbench）把每一条规则转化为智能体可以在运行时检查、审阅者可以事后验证的东西。

**Type:** Build
**Languages:** Python (stdlib)
**Prerequisites:** Phase 14 · 32 (Minimal Workbench)
**Time:** ~50 minutes

## Learning Objectives

- 把路由散文与操作性规则分离开。
- 将启动规则、禁止动作、完成定义（definition of done）、不确定性处理以及审批边界表达为机器可检查的约束。
- 实现一个规则检查器，用规则集对一次运行进行评分。
- 让规则集对 diff 友好，以便审阅时能看出改了什么。

## The Problem

典型的 `AGENTS.md` 读起来像入职文档。它告诉智能体要"小心"、"充分测试"、"不确定就问"。三天后，智能体提交了一个没有测试的改动，写入了一个被禁止的目录，并且从不发问，因为它根本不知道边界在哪里。

当指令是操作性的时，它强大；当指令是愿景性的时，它无力。解决办法是写出工作台能够解释、审阅者能够评分的规则。

## The Concept

规则应当放在 `docs/agent-rules.md` 里，远离简短的根路由。每条规则都有一个名称、一个类别和一个检查。

```mermaid
flowchart LR
  Router[AGENTS.md] --> Rules[docs/agent-rules.md]
  Rules --> Checker[rule_checker.py]
  Checker --> Report[rule_report.json]
  Report --> Reviewer[审阅者]
```

### 覆盖大多数规则的五个类别

| Category | Question the rule answers | Example |
|----------|---------------------------|---------|
| 启动 | 工作开始前必须为真的是什么？ | "state file exists and is fresh" |
| 禁止 | 什么绝不能发生？ | "do not edit `scripts/release.sh`" |
| 完成定义 | 什么能证明任务已完成？ | "pytest exits 0 and acceptance line passes" |
| 不确定性 | 智能体不确定时该怎么做？ | "open a question note instead of guessing" |
| 审批 | 什么需要人工审批？ | "any new dependency, any prod write" |

一条无法归入这五类之一的规则，通常想要成为两条规则。强制拆分它。

### 规则是机器可读的

每条规则都有一个 slug、一个类别、一行描述，以及一个 `check` 字段，该字段命名 `rule_checker.py` 中的一个函数。添加规则就意味着添加一个检查；检查器随工作台一起成长。

### 规则对 diff 友好

规则存放在单个 markdown 文件中，每条规则占一个标题。重命名在 diff 中可见。新规则放在其类别的顶部。过时的规则被删除，而不是注释掉，因为工作台才是真相来源，而不是团队上个季度感受如何的聊天记录。

### 规则与框架护栏的对比

框架护栏（OpenAI Agents SDK guardrails、LangGraph interrupts）在运行时层面强制执行规则。本课中的规则集是那些护栏所实现的、人类可读且可审阅的契约。两者你都需要：运行时在一个回合中捕获违规，规则集证明运行时在做正确的事。

### 渐进式披露：一张地图，而非一部百科全书

`AGENTS.md` 不断膨胀的原因是每次事故都会加一条规则，却没有事故移除一条。一年下来，这个文件变成两千行，而智能体读完第一屏，注意力预算就用尽了，于是只对所被告知内容的一小部分采取行动。一个巨大的指令文件失败的原因，和一份四十页的入职文档失败的原因相同：读者粗略浏览一遍，再也不回到真正重要的那部分。

解决办法不是更短的文件，而是分层的文件。根路由保持足够小，每个会话都能读完，且只包含指针。深度内容放在主题文件里，智能体只在任务触及它们时才加载。给智能体一张地图，而不是整部百科全书，让它走到所需的那一页。

```
AGENTS.md                  # router, < 50 lines: what this repo is, where to look, the 5 hard rules
docs/
  agent-rules.md           # the full rule set (this lesson)
  architecture.md          # loaded when the task touches module boundaries
  testing.md               # loaded when the task writes or runs tests
  deploy.md                # loaded only for release work, gated behind an approval rule
feature_list.json          # the backlog (Phase 14 · 36)
```

| Tier | Lives in | Read when | Size budget |
|------|----------|-----------|-------------|
| 路由 | `AGENTS.md` | 每个会话，始终 | 不超过约 50 行 |
| 规则 | `docs/agent-rules.md` | 每个会话，在启动时 | 每个类别一屏 |
| 主题文档 | `docs/<topic>.md` | 仅当任务触及该主题时 | 需要多深就多深 |

两项测试让分层保持诚实。可达性测试：智能体应当能从路由出发，最多两跳就到达任何一条规则，因此路由必须按路径链接到每个主题文档，而不是用散文描述它。新鲜度测试：路由要短到让审阅者在每次 PR 时都重读一遍，这是唯一能阻止它悄悄重新长回它所取代的那部百科全书的办法。一个不再能解析的指针，比一条缺失的规则是更糟糕的失败，因此路由中的一个断链本身就是一次启动检查违规。

## Build It

`code/main.py` 提供：

- 一个 `agent-rules.md` 解析器，把规则加载到一个 dataclass 中。
- 若干 `rule_checker.py` 风格的检查器函数，每个对应一个 `check` 引用。
- 一次演示性的智能体运行，它违反了两条规则，以及一次能捕获它们的检查通过流程。

运行它：

```
python3 code/main.py
```

输出：解析后的规则集、运行轨迹、每条规则的通过/失败，以及保存在脚本旁边的 `rule_report.json`。

## Production patterns in the wild

三种模式区分了能撑过一个季度的规则集与一周内就衰败的规则集。

**写入时打严重级标签。** 每条规则都带有 `severity`：`block`、`warn` 或 `info`。检查器报告全部三种；运行时只在 `block` 时拒绝。多数团队早期会高估严重级，然后在截止期压力下悄悄削弱它；在写入时打标签强迫提前做好校准。配合验证闸门（Phase 14 · 38）一起使用，它会把对 `block` 规则的任何越权签入一个 `overrides.jsonl` 审计日志。

**规则过期作为一种倒逼机制。** 每条规则都带有一个 `expires_at` 日期（默认为编写后 90 天）。当一条未过期的规则连续 60 天零违规时，检查器会发出警告；下一次季度审查要么为保留它给出理由，要么把它削弱为 `info`，要么删除它。Cloudflare 的生产 AI Code Review 数据（2026 年 4 月，30 天内跨 5,169 个仓库的 131,246 次审查运行）显示，带有显式过期的规则集每个仓库保持在 30 条规则以下；没有过期的规则集增长到 80 条以上，且大多数从不触发。

**Markdown 为源，JSON 为缓存。** `agent-rules.md` 是被编写的文件；`agent-rules.lock.json` 是检查器在热路径中读取的缓存。该 lock 由一个 pre-commit hook 重新生成。Markdown diff 是可审阅的；JSON 解析则不出现在每个回合中。与 `package.json` / `package-lock.json` 以及 `Cargo.toml` / `Cargo.lock` 形态相同。

## Use It

在生产中：

- Claude Code、Codex、Cursor 在会话开始时读取规则，并在拒绝动作时引用它们。检查器在 CI 中重新运行它们，以捕获无声的漂移。
- OpenAI Agents SDK guardrails 把相同的检查注册为输入护栏和输出护栏。markdown 是文档面，SDK 是运行时面。
- LangGraph interrupts 在飞行中的节点违反规则时触发。中断处理器读取规则、询问人类，然后恢复。

该规则集在这三者之间可移植，因为它只是 markdown 加函数名。

## Ship It

`outputs/skill-rule-set-builder.md` 会访谈一位项目负责人，把他们现有的散文指令归类到五个类别中，并产出一个带版本的 `agent-rules.md` 以及一个检查器桩（stub）。

## Exercises

1. 如果你的产品确实需要，添加第六个类别。论证它为何不会坍缩进五个类别之一。
2. 扩展检查器，让一条规则可以携带一个严重级（`block`、`warn`、`info`），并让报告相应地聚合。
3. 把检查器接入 CI：如果某条 block 严重级的规则在最近一次智能体运行中失败，就让构建失败。
4. 为每条规则添加一个"过期"字段。90 天没有检查失败后，该规则进入待审查状态。
5. 找一个真实的 `AGENTS.md`，把它重写为五类别规则。它有多少行是操作性的？有多少是愿景性的？

## Key Terms

| Term | What people say | What it actually means |
|------|----------------|------------------------|
| 操作性规则 | "一条真正的指令" | 一条工作台能在运行时检查的规则 |
| 愿景性规则 | "小心点" | 一条没有检查的规则；要么删除，要么升级 |
| 完成定义 | "验收" | 一个客观的、有文件支撑的、证明任务已完成的证据 |
| Block 严重级 | "硬规则" | 违规会中止运行；没有操作员就无法被消音 |
| 规则过期 | "过时规则清扫" | 一条在 N 天内没有失败的规则进入待退役状态 |

## Further Reading

- [OpenAI Agents SDK guardrails](https://platform.openai.com/docs/guides/agents-sdk/guardrails)
- [LangGraph interrupts](https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/breakpoints/)
- [Anthropic, Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)
- [Rick Hightower, Agent RuleZ: A Deterministic Policy Engine](https://medium.com/@richardhightower/agent-rulez-a-deterministic-policy-engine-for-ai-coding-agents-9489e0561edf) — 生产中的 block/warn/info 严重级
- [Cloudflare, Orchestrating AI Code Review at Scale](https://blog.cloudflare.com/ai-code-review/) — 13.1 万次审查运行，规则组合的经验教训
- [microservices.io, GenAI development platform — part 1: guardrails](https://microservices.io/post/architecture/2026/03/09/genai-development-platform-part-1-development-guardrails.html) — 规则与 CI 之间的纵深防御
- [Type-Checked Compliance: Deterministic Guardrails (arXiv 2604.01483)](https://arxiv.org/pdf/2604.01483) — Lean 4 作为规则即检查的上界
- [logi-cmd/agent-guardrails](https://github.com/logi-cmd/agent-guardrails) — 合并闸门实现：作用域、变异测试、违规预算
- Phase 14 · 32 — 本规则集落入其中的最小工作台
- Phase 14 · 38 — 消费规则报告的验证闸门
- Phase 14 · 39 — 为规则合规打分的审阅者智能体
