---
name: skill-library-zh
description: 生成一个 Voyager 形态的技能库，支持注册、按相似度检索、组合式执行以及失败驱动的精炼。
version: 1.0.0
phase: 14
lesson: 10
tags: [voyager, skills, library, composition, refinement]
---

给定一个目标运行时和一个领域，产出一个技能库，支持 Voyager 的三个组件：课程钩子、可检索的技能存储、迭代精炼。

产出：

1. `Skill` 类型，包含 `name`、`description`、`code`、`version`、`tags`、`depends_on`、`history`。每次写入都记录先前的代码。
2. `SkillLibrary`，包含 `register(skill, dedup=True)`（新建或版本号递增）、`search(query, top_k, tag_filter)`、`get(name)`、`topo_order(name)`（依赖解析）、`execute(name, context)`（拓扑顺序运行）。
3. 检索必须使用嵌入相似度或 BM25，而不是对整个库做 LLM 打分。允许在 top-k 候选短名单上做 LLM 重排。
4. 执行必须对每个技能逐一捕获异常，并将其浮现到 trace 中，作为精炼循环可以消费的反馈。
5. 一个精炼钩子：在 `execute` 失败后，运行时收集 (task, skill_name, error, env_state)，将其传给模型，并对重写后的技能调用 `register`。版本号递增；history 保留旧代码。

硬性拒绝：

- 技能是散文字符串而非代码的库。技能是可执行的。散文应放在 `description` 中。
- 没有拓扑排序的组合。深度优先且不做环检测会在技能 DAG 上崩溃。
- 静默的版本覆盖。每次精炼都必须递增 `version` 并将旧代码推入 `history` 以供审计。

拒绝规则：

- 如果目标运行时没有用于技能执行的沙箱，对于技能会触及生产系统的领域，应予以拒绝。上线前要求具备沙箱（第 09 课的原则）。
- 如果用户要求“每次失败都自动重试而不精炼”，应予以拒绝。没有精炼的重试只会放大 bug；它们并不能修复 bug。
- 如果库在扁平检索下超过约 200 个技能，拒绝将其称为“生产就绪”。先添加标签过滤和分层命名空间。

输出：`skill.py`、`library.py`、`execute.py`、`refine.py`，以及一个 `README.md`，说明去重规则、检索后端、精炼提示词和版本策略。以“接下来读什么”结尾，指向第 17 课的 Claude Agent SDK 集成、第 16 课的 OpenAI Agents SDK 工具转换，或第 30 课的技能库质量评估。
