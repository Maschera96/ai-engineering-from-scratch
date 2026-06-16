---
name: minimal-workbench-zh
description: 为任意仓库搭建三文件的最小可行智能体 workbench——简短的 AGENTS.md 路由文件、持久化的 agent_state.json，以及与项目当前 backlog 对应的 JSON task_board.json。
version: 1.0.0
phase: 14
lesson: 32
tags: [workbench, agents-md, state, task-board, scaffold]
---

给定一个仓库路径和一份简短的 backlog，搭建最小可行的智能体 workbench。

产出：

1. `AGENTS.md`，长度不超过 80 行。它必须路由到：状态文件、任务看板、更深入的规则文档（即使为空），以及验证命令。本文件中不要写任何长篇教程。
2. `agent_state.json`，包含以下键：`active_task_id`、`touched_files`、`assumptions`、`blockers`、`next_action`。所有可选字段默认为空数组或空字符串，数组绝不为 `null`。
3. `task_board.json`，为一个任务的 JSON 数组。每个任务都有 `id`、`goal`、`owner`（`builder` | `reviewer` | `human`）、`acceptance`（字符串列表）以及 `status`（`todo` | `in_progress` | `done` | `blocked`）。
4. `docs/agent-rules.md` 占位文件，每个面向维度（surface）一个 H2，方便后续课程填充。

硬性拒绝项：

- `AGENTS.md` 超过 80 行或少于 10 行。太长智能体会跳过它；太短则承载不了路由信息。
- 状态文件引用聊天记录而非仓库。仓库才是记录的权威来源（system of record）。
- 没有 `acceptance` 的任务看板。没有验收标准的任务会沦为“看着不错”的橡皮图章。
- `owner` 为 `agent` 或 `model` 的任务。owner 是角色，不是实体。

拒绝规则：

- 如果仓库没有验证命令，在提供或桩接（stub）一个之前，拒绝写入 `AGENTS.md`。路由指向一个缺失的门禁，比没有路由更糟糕。
- 如果 backlog 有超过 12 个未完成任务，拒绝并要求用户拆分。超过一屏的看板会滑向计划表演（planning theater）。
- 如果项目在被跟踪的文件中携带密钥，拒绝写入状态文件，并首先把密钥泄露作为阻塞性发现暴露出来。

输出结构：

```
<repo>/
├── AGENTS.md
├── agent_state.json
├── task_board.json
└── docs/
    └── agent-rules.md
```

以“接下来读什么”结尾，指向：

- 第 33 课，把规则占位文件变成可执行的约束。
- 第 34 课，持久化状态的 schema。
- 第 36 课，每个任务的范围契约。
