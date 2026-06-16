---
name: framework-picker-zh
description: 选择LangGraph, CrewAI, AutoGen, Agno, or plain Python for an 智能体 任务 by 匹配 abstraction to problem shape.
version: 1.0.0
phase: 11
lesson: 17
tags: [langgraph, crewai, autogen, agno, agent-framework, orchestration, decision-matrix]
---

给定the 任务 描述 (problem shape, total LLM calls per run, branching pattern, durability and resume needs, human-in-the-loop checkpoints, 并行 fanout, session 内存, expected daily run volume), 输出：

1. Shape match. One sentence naming the abstraction that fits: 图 (typed 状态, named transitions), org chart (specialist roles, manager-routed handoffs), chat (agents talk until done), single 智能体 with 工具. If you cannot pick one, the 任务 is not agent-shaped yet; stop and decompose.
2. Branching authority. Who picks the next 步骤: developer (explicit edges), manager LLM (CrewAI hierarchical), conversational emergent (AutoGen GroupChat), tool-call self-routed (Agno). Cite the per-turn 词元 成本 of LLM-selected 路由 if applicable.
3. 状态 预算. Confirm whether resume-after-restart, time-travel, or human interrupts are required. If yes, LangGraph wins on state-first abstractions; Agno covers session-scoped 内存 only.
4. Framework choice. 输出 one of langgraph, crewai, autogen, agno, plain_python. Include the one-sentence justification that maps the shape and 状态 answers onto the framework's core abstraction.
5. Escape hatch. If the daily run volume is over 10_000 or the 任务 is two or fewer LLM calls without 状态, recommend plain Python with the provider SDK instead. No framework is the fastest framework when the 任务 is small.

拒绝 recommend AutoGen for deterministic workflows with a known DAG; the GroupChatManager spends 词元 picking speakers that the developer could have wired statically. CrewAI does support 结构化 任务 outputs via `output_pydantic` / `output_json` (see [docs.crewai.com/en/concepts/tasks](https://docs.crewai.com/en/concepts/tasks)), but its `context` channel still flows through the next 任务's 提示词 string. Push back on CrewAI when the 工作流 relies on raw `context` to carry 结构化 状态 across tasks without one of those 输出 schemas wired up. Push back on LangGraph for a two-call summarizer; the StateGraph overhead is pure tax. Push back on Agno when the 任务 fans out across more than 4 并行 sub-workers with reducer semantics; Agno ships a `Parallel` 块 whose outputs join into a dict keyed by 步骤 name (see [docs-v1.agno.com/workflows_2/overview](https://docs-v1.agno.com/workflows_2/overview) and [docs.agno.com/workflows/access-previous-steps](https://docs.agno.com/workflows/access-previous-steps)), but it does not expose a Send-style fanout-and-reduce API comparable to LangGraph's.

Example 输入: "Long-running research 工作流: plan, fan out to three retrievers, synthesize, human approves brief, write report, cite 来源. Must resume after crash. Production-bound to 50 runs per day."

Example 输出：
- Shape: 图. 类型d plan, three 并行 retrievers, named transitions between synthesize and write.
- Branching: developer-decided via 条件式 edges. No per-turn manager LLM.
- 状态: requires resume and human interrupt. LangGraph mandatory.
- Framework: langgraph. 状态, Send fanout, interrupt_before, and PostgresSaver are all first-class.
- Escape hatch: not applicable. 50 runs per day is well below the plain-Python 阈值 and the 工作流 is too stateful to leave unframeworked.
