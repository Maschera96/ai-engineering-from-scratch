---
name: rule-set-builder-zh
description: 访谈项目负责人，将其现有的散文式指令归类到五个操作类别中，并生成一份带版本号的 agent-rules.md 以及一个 Python 检查器骨架。
version: 1.0.0
phase: 14
lesson: 33
tags: [rules, instructions, constraints, checker, workbench]
---

给定一个仓库以及任何现有的散文式指令（`AGENTS.md`、`CONTRIBUTING.md`、新人上手文档），生成一套工作台（workbench）可执行的五类规则集。

五个类别：

1. `startup` —— 在工作开始前必须为真的条件。
2. `forbidden` —— 绝不允许发生的事情。
3. `definition_of_done` —— 证明任务已完成的依据。
4. `uncertainty` —— 智能体在不确定时应做什么。
5. `approval` —— 需要人工签字确认的事项。

产出：

1. `docs/agent-rules.md`，每条规则一个 `##` 标题。每条规则包含 `category`、`check` 以及一行描述。
2. `tools/rule_checker.py`，包含一个 `RuleChecker` 类，为每个 `check` 暴露一个方法。每个方法接收一个 `TurnTrace` 数据类（dataclass）并返回 `bool`。
3. `tools/rule_report.py` 运行器，加载规则，在一条 trace 上运行检查器，并生成 `rule_report.json`。
4. 一份迁移说明文件：哪条散文式语句变成了哪条规则，哪些因为只是愿景性表述而被丢弃，以及原因。

硬性拒绝：

- 缺少 `check` 字段的规则。仅有愿景性的规则应放在新人上手文档中，而不是工作台规则集中。
- 单条"小心点"式的规则。请指定一个类别和一个 check，否则就移除它。
- 需要 LLM 调用的检查。规则检查必须是确定性且廉价的，以便每一轮都能运行。
- 超过 200 行的规则文件。按类别拆分为 `agent-rules.{startup,forbidden,done,uncertainty,approval}.md`，并从一个父级索引进行路由。

拒绝规则：

- 如果智能体产品无法提供 `TurnTrace`（没有埋点检测），在至少记录了 `read_state_file`、`edited_files` 和 `tests_exit_code` 之前，拒绝接入检查器。
- 如果现有指令大多是愿景性的（>50%），在生成规则前先提出这一发现。规则集看起来会很单薄；这是正确的。
- 如果某条规则是因为某一次过去的事件而添加的，附上该事件 id，以便未来复审时判断它是否仍有必要。

输出结构：

```
<repo>/
├── docs/
│   └── agent-rules.md
├── tools/
│   ├── rule_checker.py
│   └── rule_report.py
└── docs/migration-notes.md
```

以"接下来读什么"结尾，指向：

- 第 36 课，了解扩展 forbidden 类别的按任务作用域契约。
- 第 38 课，了解消费规则报告的验证关卡。
- 第 39 课，了解为规则合规性打分的审查者智能体。
