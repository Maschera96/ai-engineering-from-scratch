# LangGraph — 状态 Machines for Agents

> 一个ReAct 循环 written by hand is a `while True`. A ReAct 循环 written in LangGraph is a 图 you can checkpoint, interrupt, branch, and time-travel through. The 智能体 hasn't changed. The harness around it has.

**类型：** Build
**语言：** Python
**先修：** Phase 11 · 09 (函数调用), Phase 11 · 14 (模型 上下文 协议)
**时间：** 约 75 分钟

## 问题

你ship a function-calling 智能体. It works for three turns, then something goes wrong: the 模型 tries a 工具 that returns 500, the 用户 changes their mind mid-task, or the 智能体 decides to refund an order without a human signing off. The `while True:` 循环 has no hooks. You can't pause it, you can't rewind it, and you can't branch off into "what if the 模型 had picked the other 工具." The moment you ship this past a demo, the 智能体 becomes a black box that either worked or didn't.

这个next 步骤 is obvious once you see it. The 智能体 is already a 状态 machine — 系统 提示词 plus 消息 history plus pending 工具 calls plus the next 动作. Make the 状态 machine explicit: nodes for "the 模型 thinks," "a 工具 runs," "a human approves," and edges for the 条件式 transitions between them. Once the 图 is explicit, the harness gets four things for free: checkpointing (save 状态 between 步骤), interrupts (pause for a human), streaming (stream 词元 and intermediate events), and time-travel (rewind to a 先验 状态 and try a different branch).

LangGraph is the library that ships this abstraction. It is not an 智能体 framework in the LangChain sense ("here is an AgentExecutor, good luck"). It is a 图 runtime with first-class 状态, first-class persistence, and first-class interrupts. The 智能体 循环 is something you draw, not something you hand-write.

## 概念

![LangGraph StateGraph: nodes, edges, and the checkpointer](../assets/langgraph-stategraph.svg)

一个`StateGraph` has three things.

1. **状态.** A typed dict (类型dDict or Pydantic 模型) that flows through the 图. Every 节点 receives the full 状态 and returns a partial update, which LangGraph merges using a *reducer* per field — `operator.add` for lists that should accumulate, overwrite by default.
2. **Nodes.** Python 函数 `state -> partial_state`. Each is a discrete 步骤: "call the 模型," "run 工具," "summarize."
3. **Edges.** Transitions between nodes. Static edges go one place. 条件式 edges take a 路由器 函数 `state -> next_node_name` so the 图 can branch on 模型 输出.

你compile the 图. Compile binds the 拓扑, attaches a checkpointer (optional but essential for 生产), and returns a runnable. You invoke it with an initial 状态 and a `thread_id`. Every 步骤 of execution persists a checkpoint keyed on `(thread_id, checkpoint_id)`.

### The four superpowers

**Checkpointing.** Every 节点 transition writes the new 状态 to a store (in-memory for tests, Postgres/Redis/SQLite for prod). Resume by calling the 图 again with the same `thread_id`. The 图 picks up where it paused.

**Interrupts.** Mark a 节点 with `interrupt_before=["human_review"]` and execution stops before that 节点 runs. The 状态 persists. Your API responds to the 用户 with "awaiting approval." A later request to the same `thread_id` with `Command(resume=...)` resumes execution.

**Streaming.** `graph.stream(state, mode="updates")` yields 状态 deltas as they happen. `mode="messages"` streams the LLM 词元 inside 模型 nodes. `mode="values"` yields full snapshots. You pick what to surface in your UI.

**时间-travel.** `graph.get_state_history(thread_id)` returns the full checkpoint log. Pass any 先验 `checkpoint_id` to `graph.invoke` and you fork from that point. Great for 调试 ("what if the 模型 had picked 工具 B instead?") and for 回归 tests that replay 生产 traces.

### Reducers are the point

每个状态 field has a reducer. Most defaults are fine — a new value overwrites the old. But 消息 lists need `operator.add` so new 消息 append instead of replacing. 并行 edges merge their updates through the reducer. If two nodes both update `messages` and you forgot the `Annotated[list, add_messages]`, the second wins silently and you lose half the turn. The reducer is the only subtle thing in the library; get it right and the rest composes.

### The ReAct 图 in four nodes

一个生产 ReAct 智能体 is four nodes and two edges:

1. `agent` — calls the LLM with the current 消息 history. Returns the 助手 消息 (which may contain tool_calls).
2. `tools` — executes any tool_calls in the last 助手 消息, appends the 工具 results as 工具 消息.
3. 一个条件式 边 from `agent` that routes to `tools` if the last 消息 has tool_calls, else to `END`.
4. 一个static 边 from `tools` back to `agent`.

那is it. You get the full ReAct 循环 (思考 → 动作 → Observation → 思考 → …) with checkpointing, interrupts, and streaming, in roughly 40 lines of code.

### StateGraph vs Send (fanout)

`Send(node_name, state)` lets a 节点 dispatch 并行 subgraphs. Example: the 智能体 decides to 查询 three retrievers at once. Each `Send` spawns a 并行 execution of the 目标 节点; their outputs merge through the 状态 reducer. This is how LangGraph expresses the orchestrator-workers pattern without threading primitives.

### Subgraphs

一个compiled 图 can be a 节点 in another 图. The outer 图 sees a single 节点; the inner 图 has its own 状态 and its own checkpoints. This is how teams build supervisor-worker agents: the supervisor 图 routes 用户 intent to a per-domain worker subgraph.

## 动手构建

### 步骤 1: 状态 and nodes

```python
from typing import Annotated, TypedDict
from langchain_core.messages import AnyMessage, HumanMessage, AIMessage
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode
from langgraph.checkpoint.memory import MemorySaver

class State(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]

def agent_node(state: State) -> dict:
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

def should_continue(state: State) -> str:
    last = state["messages"][-1]
    return "tools" if getattr(last, "tool_calls", None) else END

tool_node = ToolNode(tools=[search_web, read_file])

graph = StateGraph(State)
graph.add_node("agent", agent_node)
graph.add_node("tools", tool_node)
graph.set_entry_point("agent")
graph.add_conditional_edges("agent", should_continue, {"tools": "tools", END: END})
graph.add_edge("tools", "agent")

app = graph.compile(checkpointer=MemorySaver())
```

`add_messages` is the reducer that makes the 消息 list accumulate instead of overwrite. Forgetting it is the most common LangGraph bug.

### 步骤 2: run with a thread

```python
config = {"configurable": {"thread_id": "user-42"}}
for event in app.stream(
    {"messages": [HumanMessage("find the Anthropic headquarters address")]},
    config,
    stream_mode="updates",
):
    print(event)
```

每个update is a dict `{node_name: state_delta}`. Your frontend can stream these to the UI so users see "智能体 is thinking… calling search_web… got result… 回答."

### 步骤 3: add a human-in-the-loop interrupt

Mark a 节点 so execution pauses before it runs.

```python
app = graph.compile(
    checkpointer=MemorySaver(),
    interrupt_before=["tools"],  # pause before every tool call
)

state = app.invoke({"messages": [HumanMessage("delete the production database")]}, config)
# state["__interrupt__"] is set. Inspect proposed tool calls.
# If approved:
from langgraph.types import Command
app.invoke(Command(resume=True), config)
# If denied: write a rejection message and resume
app.update_state(config, {"messages": [AIMessage("Blocked by human reviewer.")]})
```

这个状态, the checkpoint, and the thread all persist across the interrupt. Nothing is in 内存 except during execution.

### 步骤 4: time-travel for 调试

```python
history = list(app.get_state_history(config))
for snapshot in history:
    print(snapshot.values["messages"][-1].content[:80], snapshot.config)

# Fork from a prior checkpoint
target = history[3].config  # three steps back
for event in app.stream(None, target, stream_mode="values"):
    pass  # replay from that point forward
```

Passing `None` as the 输入 replays from the given checkpoint; passing a value appends it as an update to that checkpoint's 状态 before resuming. This is how you reproduce a bad 智能体 run without re-running the whole conversation.

### 步骤 5: swap the checkpointer for 生产

```python
from langgraph.checkpoint.postgres import PostgresSaver

with PostgresSaver.from_conn_string("postgresql://...") as checkpointer:
    checkpointer.setup()
    app = graph.compile(checkpointer=checkpointer)
```

SQLite, Redis, and Postgres are shipped. `MemorySaver` is for tests. Anything that persists across restarts wants a 真实 store.

## The Skill

> 你build agents as graphs, not as `while True` loops.

Before you reach for LangGraph, do a 60-second design:

1. **Name the nodes.** Every discrete decision or side-effecting 动作 is a 节点. "智能体 thinks," "工具 runs," "reviewer approves," "响应 streams." If you can't list them, the 任务 is not agent-shaped yet.
2. **Declare the 状态.** Minimal 类型dDict with a reducer for every list field. Do not stuff everything into `messages`; hoist task-specific fields (a working `plan`, a `budget` counter, a `retrieved_docs` list) to the top level.
3. **Draw the edges.** Static unless the next 步骤 depends on 模型 输出. Every 条件式 边 needs a 路由器 函数 with named branches.
4. **Choose a checkpointer up front.** `MemorySaver` for tests, Postgres/Redis/SQLite for anything else. Do not ship without one — no checkpointer means no resume, no interrupt, no time-travel.
5. **Decide interrupts before 工具 run, not after.** Approvals go on the 边 into a side-effecting 节点 so you can cancel before harm; 验证 goes on the 边 out of the 模型 so you can reject bad calls cheaply.
6. **Stream by default.** `mode="updates"` for the UI, `mode="messages"` for token-level streaming inside 模型 nodes, `mode="values"` for full snapshots during 评估.

拒绝 ship a LangGraph 智能体 that has no checkpointer. 拒绝 ship one that interrupts *after* the side effect. 拒绝 ship a `messages` field without `add_messages` as its reducer.

## 练习

1. **Easy.** Implement the four-node ReAct 图 above with a calculator 工具 and a web-search 工具. Verify that `list(app.get_state_history(config))` returns at least four checkpoints for a two-turn conversation.
2. **Medium.** Add a `planner` 节点 that runs before `agent` and writes a 结构化 `plan: list[str]` into 状态. Have `agent` mark plan 步骤 as done. Fail the test if `plan` is lost across a checkpoint resume (wrong reducer).
3. **Hard.** Build a supervisor 图 that routes between three subgraphs (`researcher`, `writer`, `reviewer`) using `Send`. Each subgraph has its own 状态 and checkpointer. Add an `interrupt_before=["writer"]` on the outer 图 so a human can approve the research brief. Confirm that time-travel from a 先验 checkpoint re-runs only the forked branch.

## Key Terms

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|StateGraph|"The LangGraph 图"|The builder object you add nodes and edges to before compile.|
|Reducer|"How the field merges"|A 函数 `(old, new) -> merged` applied when a 节点 returns an update for that field; default is overwrite, `add_messages` appends.|
|Thread|"A conversation ID"|A `thread_id` string that scopes all checkpoints for one session.|
|Checkpoint|"A paused 状态"|A persisted snapshot of the full 图 状态 after a 节点 transition, keyed on `(thread_id, checkpoint_id)`.|
|Interrupt|"Pause for a human"|`interrupt_before` / `interrupt_after` stop execution at a 节点 boundary; resume with `Command(resume=...)`.|
|时间-travel|"Fork from a 先验 步骤"|`graph.invoke(None, config_with_old_checkpoint_id)` replays from that checkpoint forward.|
|Send|"并行 subgraph dispatch"|A constructor a 节点 can return to spawn N 并行 executions of a 目标 节点.|
|Subgraph|"A compiled 图 as a 节点"|A compiled StateGraph used as a 节点 in another 图; preserves its own 状态 scope.|

## 延伸阅读

- [LangGraph documentation](https://langchain-ai.github.io/langgraph/) — canonical 参考 for StateGraph, reducers, checkpointers, and interrupts.
- [LangGraph concepts: state, reducers, checkpointers](https://langchain-ai.github.io/langgraph/concepts/low_level/) — the mental 模型 this lesson uses, straight from the 来源.
- [LangGraph Persistence and Checkpoints](https://langchain-ai.github.io/langgraph/concepts/persistence/) — the detail on Postgres/SQLite/Redis stores, checkpoint namespaces, and thread IDs.
- [LangGraph Human-in-the-loop](https://langchain-ai.github.io/langgraph/concepts/human_in_the_loop/) — `interrupt_before`, `interrupt_after`, `Command(resume=...)`, and the edit-state pattern.
- [Yao et al., "ReAct: Synergizing Reasoning and Acting in Language Models" (ICLR 2023)](https://arxiv.org/abs/2210.03629) — the pattern every LangGraph 智能体 implements; read it for the 推理 trace rationale.
- [Anthropic — Building effective agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents) — which 图 shapes (链, 路由器, orchestrator-workers, evaluator-optimizer) to prefer and when.
- Phase 11 · 09 (函数调用) — the tool-call primitive every LangGraph 智能体 节点 reuses.
- Phase 11 · 14 (模型 上下文 协议) — external 工具 discovery that plugs into a LangGraph `ToolNode` via the MCP adapter.
- Phase 11 · 17 (智能体 framework 取舍) — when to pick LangGraph over CrewAI, AutoGen, or Agno.
