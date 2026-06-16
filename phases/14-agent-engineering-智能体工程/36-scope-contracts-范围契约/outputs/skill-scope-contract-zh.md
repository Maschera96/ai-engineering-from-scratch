---
name: scope-contract-zh
description: 生成针对每个任务的范围契约，包含允许/禁止的 glob、验收标准和回滚计划，外加一个 CI 就绪的、能识别 glob 的检查器，对每次智能体 diff 运行。
version: 1.0.0
phase: 14
lesson: 36
tags: [scope, contract, globs, diff-check, ci]
---

给定一个任务描述和一个仓库布局，产出一份范围契约和一个能识别 diff 的检查器。

产出：

1. 针对该任务的 `scope_contract.json`，包含以下字段：`task_id`、`goal`、`allowed_files`（globs）、`forbidden_files`（globs）、`acceptance_criteria`、`rollback_plan`、`approvals_required`。
2. `tools/scope_check.py`，它接收一个契约路径和一组被改动的文件，返回一个 `ScopeReport`，并在任何违规时以非零状态退出。
3. CI 步骤（`.github/workflows/scope-check.yml` 或等价文件），针对合并 diff 运行该检查器。
4. `outputs/scope/closed/<task_id>.json` 归档约定，使契约随变更历史一同发布。

硬性拒绝：

- 缺少 `forbidden_files` 的契约。负空间也是契约的一部分。
- 对代码目录列出原始路径而非 glob 的契约。重构会在一夜之间让原始路径失效。
- 为空或写着 "see runbook" 的 `rollback_plan` 字段。把它写清楚。
- 列为 "case by case" 的审批。审批边界必须是可枚举的。

拒绝规则：

- 如果任务描述没有约束仓库的某个区域，拒绝仅凭描述就编写 `allowed_files`。要求提供该任务所在的目录。
- 如果仓库没有测试命令，在提供或打桩出一个之前，拒绝添加 `acceptance_criteria`。无法验证的契约只是一个愿望。
- 如果智能体运行时无法遵守审批边界（没有 human-in-the-loop），在发布前暴露这个缺口；蔓延到需要审批的操作上的范围扩张将是主要的失败模式。

输出结构：

```
<repo>/
├── scope_contract.json
├── outputs/scope/closed/
│   └── T-XXX.json
├── tools/
│   └── scope_check.py
└── .github/
    └── workflows/
        └── scope-check.yml
```

以"接下来读什么"结尾，指向：

- 第 37 课，了解将运行的命令链接回契约的运行时反馈。
- 第 38 课，了解消费范围报告的验证门。
- 第 39 课，了解审计已关闭契约归档的审查者智能体。
