# Long-Running Background Agents: 持久 Execution

> 生产环境 长周期智能体 do 不run in `while True`. 每个 LLM call becomes an activity 带有 检查点, retry, and replay. Temporal's OpenAI Agents SDK integration went GA March 2026. Claude Code Routines (Anthropic) runs scheduled Claude Code invocations 没有 a persistent local 进程. Sessions 暂停 on 人类-输入, survive deploys, and resume 来自 the latest 检查点 keyed by `thread_id`. Behind the new ergonomics sits an old pattern — 工作流 orchestration — 带有 one new 输入: LLM calls as non-deterministic activities that 必须 be deterministically replayed on recovery.

**Type:** Learn
**Languages:** Python (stdlib, minimal 持久-execution 说明 machine)
**Prerequisites:** Phase 15 · 10 (Permission modes), Phase 15 · 01 (Long-horizon agents)
**Time:** ~60 minutes

## 问题

Consider an 智能体 that runs for four hours. It calls three 工具, 提示词 the 用户 twice, and makes forty LLM calls. Halfway through, the host it is running on reboots. 内容 happens?

- In a naive `while True` 循环: everything is lost. The 运行 restarts 来自 scratch. The three 工具 calls (带有 real side effects) execute again. The 用户 is prompted again for things they already approved. Forty LLM calls are re-billed.
- 带有 持久 execution: the 运行 resumes 来自 the most recent 检查点. Already-completed activities are 不re-executed; their 结果 are replayed 来自 the 持久 日志. The 用户 does 不re-approve things they already approved. The LLM calls already made are 不re-billed.

这is the same pattern 工作流 engines have shipped for a decade (Temporal, 节奏, Uber's Cherami). 内容's new is that LLM calls are now a kind of activity — non-deterministic, expensive, 带有 side effects — and they fit this pattern cleanly.

这个running theme of the lesson: long-horizon 可靠性 decays (METR observes a "35-minute degradation" — success rate drops roughly quadratically 带有 horizon). 持久 execution enables runs that are longer than the 可靠性 profile supports, 哪个 is a new way to fail safely if the 设计 is right and unsafely if the 设计 is wrong.

## 概念

### Activities, 工作流s, and replay

- **工作流**: deterministic orchestration code. Defines the sequence of activities, the branches, the waits. Must be deterministic so it can be replayed 来自 the event 日志 没有 surprising divergence.
- **Activity**: a non-deterministic, potentially failing unit of work. LLM call, 工具 call, 文件 编写, HTTP request. Each activity is logged 带有 its 输入 and (once complete) its 输出.
- **Event 日志**: the 持久 backing store. 每个 activity start, complete, fail, retry, and 每个 工作流 decision is recorded.
- **Replay**: on recovery, the 工作流 code re-runs 来自 the start; 每个 activity that already completed returns its logged 结果 没有 re-executing. Only activities that had 不completed are actually 运行.

这is the same shape as React re-rendering 针对 a virtual DOM, or Git rebuilding a working tree 来自 commits. Determinism in the orchestrator is 什么 makes 持久性 cheap.

### 原因 LLM calls fit the pattern

LLM calls are:
- Non-deterministic (temperature > 0; even temperature 0 drifts across 模型 versions).
- Expensive (money and latency).
- Potentially failing (rate 限制, timeouts).
- Side-effectful (if they invoke 工具).

这is exactly the activity profile. Wrapping 每个 LLM call as an activity gives you retry 带有 exponential backoff, 检查点ing across restarts, and a replayable trace for debugging.

### Checkpoints keyed by `thread_id`

LangGraph, Microsoft Agent Framework, Cloudflare 持久 Objects, and Claude Code Routines 所有 converged on the same API shape: a `thread_id` (or equivalent) identifies the session; each 说明 transition persists to a 后端 (PostgreSQL 默认值, SQLite for dev, Redis for cache); resume reads the latest 检查点.

这个backend choice matters:

- **PostgreSQL**: 持久, 可查询, survives deploys. 默认值 for LangGraph.
- **SQLite**: local-dev 只; loses data across hosts.
- **Redis**: fast but ephemeral unless AOF/snapshot configured.
- **Cloudflare 持久 Objects**: transparently distributed; scoped by a unique key; survives for hours to weeks.

### 人类-输入 as a first-类别 说明

Propose-then-commit (Lesson 15) 要求 a 持久 "waiting on 人类" 说明. The 工作流 pauses, the 外部 queue holds the pending request, and an approval resumes 来自 exactly that point. 没有 持久性 this is best-effort; 带有 it, an overnight approval arrives and the 工作流 picks up in the morning.

### 这个35-minute degradation

METR observed that 每个 智能体 类别 measured shows 可靠性 decay beyond ~35 minutes of continuous operation. Doubling the 任务 duration roughly quadruples the 失败 rate. 持久 execution does 不fix this; it lets you 运行 longer than the 可靠性 profile supports. The safe pattern is to combine 持久性 带有 检查点s that 要求 fresh 人在回路 on re-entry, and 带有 预算 熔断开关es (Lesson 13) that 上限 total 计算 regardless of wall-clock time.

### 当durable execution is the wrong answer

- Runs shorter than a few minutes 带有 no 人类 输入. Overhead > benefit.
- Strictly 阅读-只 information retrieval.
- 任务 在哪里 正确性 要求 end-to-end within one 上下文 window (some reasoning 任务; some one-shot generation).

```figure
memory-consolidation
```

## 使用它

`code/main.py` implements a minimal 持久-execution engine in stdlib Python. It supports:

- `@activity` decorator that 日志 输入 and 输出 to a JSON event 日志.
- A 工作流 function that sequences activities.
- A `run_or_replay(workflow, event_log)` function that replays completed activities 没有 re-executing them.

这个driver simulates a three-activity 工作流, crashes halfway through, and shows (a) a naive retry re-executing everything versus (b) a replay running 只 the 缺失 activity.

## 交付它

`outputs/skill-durable-execution-review.md` reviews a 拟议的 long-running 智能体 部署 for correct 持久-execution shape: activities, determinism, 检查点 后端, 人类-输入 说明, and 人在回路-on-resume 政策.

## 练习

1. 运行 `code/main.py`. Observe the difference in activity-execution count between naive retry and replay. Change the crash point and show the replay count changes accordingly.

2. Convert the toy engine to use `thread_id` explicitly. Simulate two concurrent sessions sharing the engine and 确认 their event 日志 do 不collide.

3. Take one activity in the toy engine. Introduce a non-determinism (a wall-clock timestamp 内部 a 工作流 decision). Demonstrate the divergence on replay. 解释 如何 real engines handle this (side-effect registration, `Workflow.now()` APIs).

4. 阅读 the LangChain "Runtime behind 生产环境 deep agents" post. 列出 每个 说明 that the runtime persists and 命名 哪个 失败 mode each covers.

5. 设计 a 检查点 政策 for a 6-hour 自主 coding 任务. 位置 do you 检查点? 内容 does resume-on-crash look like? 内容 要求 fresh 人在回路?

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| 工作流 | "Agent's script" | Deterministic orchestration code; replayable 来自 event 日志 |
| Activity | "A step" | Non-deterministic unit (LLM call, 工具 call); logged 之前 and 之后 |
| Event 日志 | "The backing store" | 持久 record of 每个 说明 transition |
| Replay | "Resume" | Re-运行 工作流; completed activities 返回 logged 结果 没有 re-execution |
| 检查点 | "Save point" | Persisted 说明 keyed by thread_id; latest-wins on resume |
| thread_id | "Session key" | Identifier that scopes 持久 说明 |
| 35-minute degradation | "Reliability decay" | METR: success rate drops ~quadratically 带有 horizon |
| Non-determinism | "Drift on replay" | Wall clock, random, LLM 输出; 必须 be registered as side effect |

## 延伸阅读

- [Anthropic — Claude Code Agent SDK: agent loop](https://code.claude.com/docs/en/agent-sdk/agent-loop) — 预算, turns, and resume semantics.
- [Microsoft — Agent Framework: human-in-the-loop and checkpointing](https://learn.microsoft.com/en-us/agent-framework/workflows/human-in-the-loop) — RequestInfoEvent shape.
- [LangChain — The Runtime Behind Production Deep Agents](https://www.langchain.com/conceptual-guides/runtime-behind-production-deep-agents) — 具体 runtime requirements.
- [OpenAI Agents SDK + Temporal integration (Trigger.dev announcement)](https://trigger.dev) — activity shape for LLM calls.
- [Anthropic — Measuring agent autonomy in practice](https://www.anthropic.com/research/measuring-agent-autonomy) — the 35-minute degradation reference.
