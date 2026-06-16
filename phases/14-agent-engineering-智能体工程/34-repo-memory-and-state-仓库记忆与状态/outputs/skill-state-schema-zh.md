---
name: state-schema-zh
description: 为智能体状态和任务看板生成项目专属的 JSON Schema、一个具备原子写入能力的 Python StateManager，以及一个迁移脚手架，使模式升级不会损坏工作台。
version: 1.0.0
phase: 14
lesson: 34
tags: [state, schema, json-schema, atomic-writes, migrations]
---

给定一个仓库以及在其中运行的智能体产品，为工作台生成模式优先（schema-first）的状态文件。

产出：

1. `schemas/agent_state.schema.json`，覆盖必需的键、允许的 status 值、数组与 null 的纪律，以及一个 `schema_version` 整数。
2. `schemas/task_board.schema.json`，覆盖任务 id 的模式、允许的负责人、允许的状态，以及验收数组。
3. `tools/state_manager.py`，暴露 `load`、`commit` 和 `update`，使用临时文件加重命名（temp-and-rename）的原子写入。
4. `tools/migrate_state.py`，为下一次模式升级提供脚手架，若文件来自未知版本则大声失败（fail-loud）。
5. `agent_state.json` 和 `task_board.json`，以 `schema_version: 1` 和全新的 backlog 进行初始化。

硬性拒绝项：

- 没有 `schema_version` 字段的模式。迁移不是可选项。
- 在应为数组的位置允许 `null`。`null` 是伪装成数据的写入期 bug。
- 使用纯 `open(path, "w")` 的写入器。只允许原子写入；不完整的文件会损坏唯一真相源。
- 在状态中存储 token、原始聊天记录或 PII。状态用于存储与仓库相关的事实。

拒绝规则：

- 如果仓库没有版本控制，拒绝交付状态文件。原子写入加 git diff 才是持久性的保障方案。
- 如果项目没有至少一个用于验证 `done` 转换的验收命令，拒绝 `status: done` 这个枚举值。在没有验收检查的情况下添加 `done` 只是作秀。
- 如果项目打算在多个进程间共享状态却没有锁策略，请在交付前指出这一发现；原子重命名是必要的但不充分。

输出结构：

```
<repo>/
├── agent_state.json
├── task_board.json
├── schemas/
│   ├── agent_state.schema.json
│   └── task_board.schema.json
└── tools/
    ├── state_manager.py
    └── migrate_state.py
```

以“接下来读什么”结尾，指向：

- 第 35 课，介绍在启动时调用该管理器的初始化脚本。
- 第 38 课，介绍读取状态以对完成情况打分的验证关卡。
- 第 40 课，介绍消费同一模式的交接生成器。
