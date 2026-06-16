---
name: reviewer-agent-zh
description: 搭建一个评审智能体角色，配备五维评分量表，读取构建者产出物，生成结构化评审报告，让人类评审从一份已写好的页面而非空白页面开始。
version: 1.0.0
phase: 14
lesson: 39
tags: [reviewer, rubric, role-separation, second-loop, review-report]
---

在已有一个产出工作台产出物的构建者智能体的前提下，搭建一个读取这些产出物并撰写结构化报告的评审者。

产出：

1. `agents/reviewer.md`，包含评审者系统提示词：只读访问、五维评分量表、必须为每项评分引用产出物路径。
2. `tools/reviewer.py`，从工作台加载 `ReviewerInputs`，并按维度运行 LLM 打分器。
3. `outputs/review/<task_id>.json`，作为评审报告的规范路径。
4. `docs/reviewer-rubric.md`，列出五个维度、每个维度回答的问题，以及 0-1-2 的锚点描述。
5. CI 步骤，在构建者任务关闭时将评审报告作为 PR 评论发布。

硬性拒绝：

- 一个对 diff 拥有写权限的评审者。构建者与评审者之间的差距才是全部信号；将其合并会摧毁可靠性。
- 一个没有逐分锚点描述的评分量表。没有锚点的“从 0 到 2 打分”会退化成凭感觉。
- 省略引用的评审报告。每个评分都必须指向某个文件或追踪条目。
- 共用构建者的系统提示词。用同一个模型可以；用同一个提示词不行。

拒绝规则：

- 如果构建者没有产出验证报告，拒绝运行评审者。在值得寻求判断之前，验收必须成立。
- 如果项目关闭的任务少于三个，拒绝声称量表已校准。把最初几份报告保存为校准集。
- 如果评审者被要求在低于最低置信度的情况下打分，拒绝，并将不确定的维度上交给人类。

输出结构：

```
<repo>/
├── agents/reviewer.md
├── tools/reviewer.py
├── outputs/review/
│   └── <task_id>.json
├── docs/reviewer-rubric.md
└── .github/workflows/review.yml
```

以“接下来读什么”结尾，指向：

- 第 40 课，了解结合验证 + 评审的交接包。
- 第 41 课，了解端到端演练构建者/评审者分离的真实风格任务。
- 第 05 课（Self-Refine 与 CRITIC），了解本课所改进的单智能体自我评审基线。
