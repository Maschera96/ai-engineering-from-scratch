# 技能库与终身学习（Voyager）

> Voyager（Wang 等人，TMLR 2024）将可执行代码视为一种技能。技能是有名称的、可检索的、可组合的，并通过环境反馈不断精炼。这是 Claude Agent SDK skills、skillkit 以及 2026 年技能库模式的参考架构。

**类型：** 构建
**语言：** Python（标准库）
**前置要求：** Phase 14 · 07（MemGPT）、Phase 14 · 08（LettBlocks）
**时长：** 约 75 分钟

## 学习目标

- 说出 Voyager 的三个组件——自动课程（automatic curriculum）、技能库（skill library）、迭代式提示（iterative prompting）——以及各自的作用。
- 解释为什么 Voyager 把动作空间设为代码，而非原始命令。
- 用 stdlib 实现一个具备注册、检索、组合和失败驱动精炼能力的技能库。
- 将 Voyager 的模式映射到 2026 年的 Claude Agent SDK skills 与 skillkit 生态。

## 问题所在

在每个会话中都从零重建每项能力的智能体，会犯三个错误：

1。**浪费 token。** 每个任务都要重新引出相同的推理。
2。**丢失进度。** 在会话 A 中学到的纠正无法迁移到会话 B。
3。**在长程组合上失败。** 复杂任务需要能力层级；一次性提示无法表达它们。

Voyager 的答案是：把每项可复用的能力视为一段有名称的代码，存放在库中，可按相似度检索，可与其他技能组合，并通过执行反馈不断精炼。

## 概念

### 三个组件

Voyager（arXiv:2305.16291）围绕以下结构来组织智能体：

1。**自动课程。** 一个由好奇心驱动的提议者，根据智能体当前的技能集和环境状态挑选下一个任务。探索是自底向上的。
2。**技能库。** 每个技能都是可执行代码。任务成功时会添加新技能。技能按查询与描述的相似度检索。
3。**迭代式提示机制。** 失败时，智能体会收到执行错误、环境反馈和自我验证输出，然后精炼该技能。

Minecraft 评估（Wang 等人，2024）：相比基线，独特物品多 3.3 倍，制造石器快 8.5 倍，制造铁器快 6.4 倍，地图穿越距离长 2.3 倍。这些数字是 Minecraft 特有的，但模式可以迁移。

### 动作空间 = 代码

大多数智能体输出原始命令。Voyager 输出 JavaScript 函数。一个技能是：

```
async function craftIronPickaxe(bot) {
  await mineIron(bot，3);
  await mineStick(bot，2);
  await placeCraftingTable(bot);
  await craft(bot，'iron_pickaxe');
}
```

由子技能组合而成。以描述和嵌入为键存储。作为程序而非提示被检索。

这就是 2026 年的 Claude Agent SDK 技能：一段有名称的、可检索的代码，加上智能体按需加载的指令。

### 技能检索

新任务「制作一把钻石镐」。智能体：

1。嵌入任务描述。
2。在技能库中查询 top-k 个相似技能。
3。检索出 `craftIronPickaxe`、`mineDiamond`、`placeCraftingTable` 等。
4。由检索到的基础技能 + 新逻辑组合出新技能。

这正是 MCP 资源（Phase 13）与 Agent SDK skills 实现的模式：在知识/代码面上进行检索，并限定到当前任务的范围。

### 迭代式精炼

Voyager 的反馈循环：

1。智能体编写一个技能。
2。技能在环境中运行。
3。返回三种信号之一：`success`、`error`（带堆栈跟踪）、`self-verification failure`。
4。智能体以该信号为上下文重写技能。
5。循环直到成功或达到最大轮数。

这是把 Self-Refine（第 05 课）应用于以环境为基准进行验证的代码生成。CRITIC（第 05 课）是同一模式，只是用外部工具作为验证器。

### 课程与探索

Voyager 的课程模块会根据智能体已有什么、尚未做什么，提议诸如「在湖边搭建庇护所」之类的任务。提议者利用环境状态 + 技能清单，挑选一个略高于当前能力的任务——探索的最佳点。

对生产环境的智能体而言，这转化为一个「缺什么」算子：给定当前技能库和一个领域，我们还没有覆盖哪些技能？团队通常手动实现为课程评审。

### 这种模式会在哪里出问题

- **技能库腐烂。** 同一个技能以略有差异的描述被添加 10 次。在写入时做去重；检索只返回一个。
- **组合技能漂移。** 父技能依赖一个被精炼过的子技能。给技能加版本；锁定到 v1 的父技能不会神奇地用上 v3。
- **检索质量。** 当库增长到几百个以上时，对技能描述做向量检索会退化。用标签过滤和硬约束（「只要 `category=tooling` 的技能」）来补充。

## 动手构建

`code/main.py` 用 stdlib 实现了一个技能库：

- `Skill` —— 名称、描述、代码（字符串形式）、版本、标签、依赖。
- `SkillLibrary` —— 注册、搜索（token 重叠）、组合（依赖的拓扑排序），以及精炼（更新时版本递增）。
- 一个脚本化的智能体，注册三个基础技能、组合出第四个、遇到一次失败并进行精炼。

运行它：

```
python3 code/main.py
```

这段执行轨迹展示了库写入、检索、组合、一次失败的执行，以及一次 v2 精炼——完整呈现 Voyager 的循环。

## 应用场景

- **Claude Agent SDK skills**（Anthropic）—— 2026 年的参考实现：每个技能有描述、代码和指令；在智能体会话期间按需加载。
- **skillkit**（npm：skillkit）—— 面向 32+ 个 AI 编码智能体的跨智能体技能管理。
- **自定义技能库** —— 领域特定（用于数据智能体的 SQL 技能，用于基础设施智能体的 Terraform 技能）。Voyager 模式也能向下伸缩。
- **OpenAI Agents SDK `tools`** —— 处于低端；每个工具都是一个轻量级技能。

## 交付落地

`outputs/skill-skill-library.md` 生成一个 Voyager 形态的技能库，为任意目标运行时接好了注册、检索、版本管理和精炼。

## 练习

1。为 `compose()` 添加一个依赖环检测器。当技能 A 依赖 B、B 又依赖 A 时会发生什么？报错还是警告？
2。实现按技能的版本锁定。当父技能组合子技能 `crafting@1` 时，对 `crafting@2` 的精炼绝不能悄悄升级父技能。
3。用 sentence-transformers 嵌入（或一个 stdlib 实现的 BM25）替换 token 重叠检索。在一个 50 技能的玩具库上测量 retrieval@5。
4。添加一个「课程」智能体：给定当前库和一个领域描述，提议 5 个缺失的技能。每周调用一次。
5。阅读 Anthropic 的 Claude Agent SDK 技能文档。把玩具库移植到 SDK 的技能模式。可发现性会有什么变化？

## 关键术语

| 术语 | 人们怎么说 | 它实际的含义 |
|------|----------------|------------------------|
| 技能（Skill） | “可复用的能力” | 有名称的代码块 + 描述，可按相似度检索 |
| 技能库（Skill library） | “智能体关于如何做的记忆” | 持久化的技能存储，可搜索、可组合 |
| 课程（Curriculum） | “任务提议者” | 由当前能力缺口驱动的自底向上目标生成器 |
| 组合（Composition） | “技能 DAG” | 技能调用技能；执行时拓扑排序 |
| 迭代式精炼（Iterative refinement） | “自我纠正循环” | 环境反馈 + 错误 + 自我验证回流进下一个版本 |
| 动作空间即代码（Action-space-as-code） | “程序化动作” | 输出函数而非原始命令，以实现时间上延展的行为 |
| 写入时去重（Dedup on write） | “技能坍缩” | 近似重复的描述坍缩为一个规范技能 |

## 延伸阅读

- [Wang et al.，Voyager (arXiv:2305.16291)](https://arxiv.org/abs/2305.16291) —— 最初的技能库论文
- [Claude Agent SDK overview](https://platform.claude.com/docs/en/agent-sdk/overview) —— 作为 2026 年产品化形态的 skills
- [Anthropic，构建ing agents withClaude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk) —— 实践中的 skills 与 subagents
- [Madaan et al.，Self-Refine (arXiv:2303.17651)](https://arxiv.org/abs/2303.17651) —— Voyager 底层的精炼循环
