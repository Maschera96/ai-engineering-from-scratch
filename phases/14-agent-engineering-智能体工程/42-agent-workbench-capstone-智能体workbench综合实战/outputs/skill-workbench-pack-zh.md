---
name: workbench-pack-zh
description: 生成一个针对项目调优的即插即用智能体 workbench 包——规则针对团队历史进行打磨，作用域 glob 匹配仓库，评分量表维度扩展一个领域专属条目。
version: 1.0.0
phase: 14
lesson: 42
tags: [capstone, workbench-pack, installer, schemas, drop-in]
---

给定一个仓库、团队的事故历史，以及运行其中的智能体产品，输出一个调优后的 agent-workbench-pack 和一个安装器。

产出：

1. `agent-workbench-pack/` 目录，符合规范布局：AGENTS.md、docs/、schemas/、scripts/、bin/、README.md、VERSION。
2. 一个 `bin/install.sh`，在没有 `--force` 的情况下拒绝覆盖已存在的包，并将 `.workbench-version` 写入目标仓库。
3. 针对项目调优的 `agent-rules.md`（每个类别至少有一条规则源自团队最近六次事故）、`reviewer-rubric.md`（含第六个领域维度）和 `scope_contract.schema.json`（含项目专属 glob）。
4. 一个 `lint_pack.py` 脚本，当脚本与模式之间、或 VERSION 与模式的 `schema_version` 之间出现漂移时失败。
5. 可选的 CI 集成，在演示分支上安装该包，并针对一个已知良好的任务运行验证关卡。

硬性拒绝：

- 包含项目专属任务的包。任务存在于目标仓库的看板上。
- 绑定单一厂商 SDK 的包。仅限框架无关；SDK 接线是目标仓库的工作。
- 修改状态文件的安装器。安装器是幂等的、仅触及表层；状态归智能体和人类所有。
- 没有对应检查函数的规则。理想化的规则属于入职培训，而非该包。

拒绝规则：

- 如果事故历史为空，拒绝交付调优后的 `agent-rules.md`。使用规范默认值并暴露这一缺口。
- 如果目标仓库的 CI 与安装不兼容（没有 `.github/workflows/`，也没有等价物），拒绝可选的 CI 步骤并记录手动路径。
- 如果团队使用该包的私有 fork，拒绝写出公共安装器。私有安装器携带私有不变量。

输出结构：

```
agent-workbench-pack/
├── AGENTS.md
├── docs/
├── schemas/
├── scripts/
├── bin/install.sh
├── lint_pack.py
├── VERSION
└── README.md
```

以“接下来读什么”结尾，指向：

- 第 41 课，本包所改进的前后对比基准。
- 第 30 课（Eval-Driven Agent Development），消费本包裁决结果的评估循环。
- [SkillKit](https://github.com/rohitg00/skillkit)，用于将该包分发到 32 个 AI 智能体。
