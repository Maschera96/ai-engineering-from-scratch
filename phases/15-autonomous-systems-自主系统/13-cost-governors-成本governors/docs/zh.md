# Action 预算, Iteration 上限, and Cost Governors

> A mid-sized e-commerce 智能体's monthly LLM 成本 jumped 来自 $1,200 to $4,800 之后 its team enabled the "order-tracking" skill. That is 不a pricing bug. That is an 智能体 that found a new 循环 and kept spending 内部 it. Microsoft's Agent Governance Toolkit (April 2, 2026) codifies the 防御 针对 this 类别: per-request `max_tokens`, per-任务 词元 and dollar 预算, per-day/month 上限, iteration 上限, tiered 模型 routing, 提示词 caching, 上下文 windowing, 人在回路 检查点s on expensive actions, 熔断开关es on 预算 breach. Anthropic's Claude Code Agent SDK ships the same primitives under different names. Financial velocity 限制 — e.g. cut access on >$50 in 10 minutes — catch 循环 faster than monthly 上限.

**Type:** Learn
**Languages:** Python (stdlib, layered cost-governor simulator)
**Prerequisites:** Phase 15 · 10 (Permission modes), Phase 15 · 12 (持久 execution)
**Time:** ~60 minutes

## 问题

自主智能体 spend real money on 每个 turn. A chatbot's bad 输出 is a bad reply; an 智能体's bad 循环 is a bill. The industry-documented term for the 失败 mode is "Denial of Wallet" — the 智能体 keeps reasoning, keeps 工具-calling, keeps billing, and nothing stops it because nothing was designed to.

这个fix is 不one number. It is a stack of 限制 at different time scales and granularities: per-request, per-任务, per-hour, per-day, per-month. A well-designed stack catches a runaway 循环 within minutes, a slow leak within hours, and a bad release within a day. The same stack keeps a 预算 at 所有 when the 智能体 is long-horizon and 自主.

这is an engineering lesson: the math is trivial, the discipline is 在哪里 teams fail. The 列出 of 限制 below is 所有 具名 either in the Microsoft Agent Governance Toolkit or the Anthropic Claude Code Agent SDK docs.

## 概念

### 这个cost-governor stack

1. **`max_tokens` per request.** Simple. Prevents 任何 one call 来自 emitting an unbounded completion.
2. **Per-任务 词元 预算.** Across the whole 运行, do 不exceed N 词元. Hard stop at the 上限.
3. **Per-任务 dollar 预算.** Same as 词元 but in currency. `max_budget_usd` in Claude Code.
4. **Per-工具 call 上限.** No more than N `WebFetch` calls, N `shell_exec` calls, etc.
5. **Iteration 上限 (`max_turns`).** Total 智能体 循环 iterations; prevents infinite reasoning 循环.
6. **Per-minute / per-hour / per-day / per-month 上限.** Rolling windows. Catches leaks at different time scales.
7. **Financial velocity 限制.** E.g., "if spend exceeds $50 in 10 minutes, cut access." Catches 循环-based burn 之前 monthly 上限 fire.
8. **Tiered 模型 routing.** 默认值 to a smaller 模型; escalate to a larger one 只 when a 分类器 judges the 任务 warrants it.
9. **提示词 caching.** System 提示词 and stable 上下文 stored in provider cache; 词元 成本 of re-sending is near zero.
10. **Context windowing.** Compaction / summarization to keep the active 上下文 below a 阈值; direct 词元-成本 reduction.
11. **人在回路 检查点s on expensive actions.** 之前 an action known to be expensive (long 工具 call, large download, a costly 模型 upgrade), 要求 a 人类 tap.
12. **Kill switch on 预算 breach.** Session aborts when 任何 上限 fires. 上限 is recorded; 要求 a separate re-enable path.

### 原因 the stack, 不one 上限

一个single monthly 上限 catches a runaway 智能体 只 之后 the wallet is gone. A 单一 per-request 上限 catches nothing at the session level. Different 失败 modes 要求 different time scales:

- **Runaway 循环** (智能体 stuck in a 5-second retry): caught by velocity 限制.
- **Slow leak** (智能体 doing ~2x expected work per 任务): caught by daily 上限.
- **Bad release** (new 版本 uses 5x 词元): caught by weekly / monthly 上限.
- **Legitimate surge** (real demand, 不a bug): caught by hour / day 上限 带有 clear 日志.

### Claude Code's 预算 表面

这个Claude Code Agent SDK exposes (public docs):

- `max_turns` — iteration 上限.
- `max_budget_usd` — dollar 上限; session aborts on breach.
- `allowed_tools` / `disallowed_tools` — 工具 allowlist and denylist.
- Hook points 之前 工具 use for custom 成本-accounting.

Combine 带有 the permission-mode ladder (Lesson 10). An `autoMode` session 没有 `max_budget_usd` is ungoverned autonomy. Anthropic explicitly frames Auto Mode as requiring 预算 控制; the 分类器 is orthogonal to 成本.

### EU AI Act, OWASP Agentic Top 10

Microsoft's Agent Governance Toolkit covers the OWASP Agentic Top 10 and the EU AI Act Article 14 (人类 oversight) requirements. For 生产环境 in the EU, 日志 and 上限 enforcement are 不optional.

### 这个observed $1,200 → $4,800 case

这个real case in the Microsoft docs: an e-commerce 智能体 whose monthly 成本 tripled 之后 a new 工具 was added. The 工具 allowed the 智能体 to poll order status 期间 每个 session. No 循环 detection. No per-工具 上限. No 告警 on week-over-week growth. The fix was a per-工具 上限 plus a daily-growth 告警. This is a template: 每个 new 工具 表面 is a new potential 循环; 每个 new 工具 needs its own 上限 and its own 告警.

## 使用它

`code/main.py` simulates an 智能体 运行 带有 and 没有 a layered 成本-governor stack. The simulated 智能体 drifts 进入 a polling 循环 之后 some turns; the layered stack catches it within the velocity window while a 单一 monthly 上限 would 不fire until days later.

## 交付它

`outputs/skill-agent-budget-audit.md` audits a 拟议的 智能体 部署's 成本-governor stack and flags 缺失 层.

## 练习

1. 运行 `code/main.py`. 确认 the velocity 限制 fires 之前 the iteration 上限 on a polling-循环 轨迹. Now disable the velocity 限制 and measure 如何 much the 智能体 "spends" 之前 the iteration 上限 catches it.

2. 设计 a per-工具 上限 设置 for a browser 智能体 (Lesson 11). Which 工具 needs the tightest 上限? Which 工具 can 运行 unbounded 没有 风险?

3. 阅读 the Microsoft Agent Governance Toolkit docs. 列出 每个 上限 type the toolkit names. 映射 each to one of the 失败 modes (runaway 循环, slow leak, bad release, surge).

4. Price an overnight unattended 运行 for a realistic 任务 (e.g., "triage 50 issues in a repo"). 设置 `max_budget_usd` at 2x your point 估计. Justify the 2x.

5. Claude Code's `max_budget_usd` fires on session aggregate 成本. 设计 a complementary velocity 限制 you would enforce externally. 内容 triggers the cut-off, and 什么 does re-enable look like?

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| Denial of Wallet | "Runaway bill" | Agent 循环 generating spend 带有 no 上限 to stop it |
| max_tokens | "Per-request 上限" | Ceiling on a 单一 completion's size |
| max_turns | "Iteration 上限" | Ceiling on 智能体 循环 iterations in a session |
| max_预算_usd | "Dollar 熔断开关" | Session 成本 上限; aborts on breach |
| Velocity 限制 | "Rate 上限" | Limit on spend per short window (e.g., $50 / 10 min) |
| Tiered routing | "Small 模型 first" | Cheap 模型 默认值; escalate 只 when 分类器 warrants |
| 提示词 caching | "Cached 系统 提示词" | Provider-side cache reduces re-send 词元 成本 to near zero |
| 人在回路 检查点 | "人类 approval gate" | 人类 tap 必需 之前 expensive action |

## 延伸阅读

- [Anthropic Claude Code Agent SDK — agent loop and budgets](https://code.claude.com/docs/en/agent-sdk/agent-loop) — `max_turns`, `max_budget_usd`, 工具 allowlists.
- [Microsoft Agent Framework — human-in-the-loop and governance](https://learn.microsoft.com/en-us/agent-framework/workflows/human-in-the-loop) — 成本-governor 检查点s.
- [Anthropic — Claude Managed Agents overview](https://platform.claude.com/docs/en/managed-agents/overview) — provider-side 成本 控制.
- [Anthropic — Prompt caching (Claude API docs)](https://platform.claude.com/docs/en/prompt-caching) — caching mechanics.
- [Anthropic — Measuring agent autonomy in practice](https://www.anthropic.com/research/measuring-agent-autonomy) — 成本 profile for 长周期智能体.
