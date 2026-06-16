---
name: handoff-generator-zh
description: 从工作台产物生成会话结束的 handoff 数据包，同时产出人类可读的 Markdown 和以七个规范字段为键的机器可读 JSON。
version: 1.0.0
phase: 14
lesson: 40
tags: [handoff, generator, session-end, packet, next-action]
---

给定一个工作台（状态、裁决、审查、反馈日志、diff），生成一个接入智能体运行时的会话结束 handoff 生成器。

产出：

1. `tools/generate_handoff.py`，暴露 `generate_handoff(snapshot) -> (markdown, payload)`。
2. `outputs/handoff/<session_id>/handoff.md` 和 `handoff.json`。
3. `handoff.schema.json`，覆盖七个必需字段以及反馈尾部格式。
4. 会话结束的 hook 脚本，运行生成器，并在任何字段缺失时拒绝关闭会话。
5. `docs/handoff.md`，列出七个字段、它们的来源以及裁剪策略。

硬性拒绝：

- 没有 `next_action` 的 handoff。伪装成 handoff 的状态报告会毒害下一个会话。
- 手写摘要的生成器。智能体的职责是把工作台留在可生成的状态。
- 与 JSON 不一致的 markdown 数据包。JSON 是源；markdown 是 JSON 的渲染。
- 超过 30 条的反馈尾部。完整日志在版本控制中；数据包必须保持精简。

拒绝规则：

- 如果验证报告缺失，拒绝生成数据包。没有裁决的 handoff 只是一个愿望。
- 如果审查报告缺失且预期有人类审查者，拒绝并要求先通过审查。
- 如果 diff 摘要为空但会话运行时间超过 5 分钟，在生成前先暴露异常；应怀疑是会话卡死而非真正的空操作。

输出结构：

```
<repo>/
├── outputs/handoff/<session_id>/
│   ├── handoff.md
│   └── handoff.json
├── tools/generate_handoff.py
├── handoff.schema.json
└── docs/handoff.md
```

以"接下来读什么"结尾，指向：

- 第 41 课，在一个仿真实的示例应用上做端到端练习。
- 第 42 课，将生成器打包进 capstone 工作台包。
- 第 29 课（Production Runtimes），将会话结束接入队列、事件和 cron 触发器。
