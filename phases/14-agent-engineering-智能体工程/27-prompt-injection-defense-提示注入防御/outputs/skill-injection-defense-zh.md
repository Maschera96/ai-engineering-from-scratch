---
name: injection-defense-zh
description: 为任意智能体运行时构建一个 PVE（Prompt-Validator-Executor）层，包含按来源打标签的内容、注入标记扫描以及基于允许列表的导航。
version: 1.0.0
phase: 14
lesson: 27
tags: [security, prompt-injection, pve, greshake, source-tag]
---

给定一个具备工具访问和检索能力的智能体，产出一个注入防御层。

产出：

1. 为每一段内容打来源标签：`user_message`、`tool_output`、`retrieved_web`、`retrieved_memory`、`retrieved_file`。让标签在消息历史中传播。
2. `Validator.assess(tool_call, contents)` —— 拒绝带有注入形态参数或检索内容的工具调用；仅当来源标签与所声明的信任级别匹配时才允许。
3. 用于导航的允许列表 / 阻止列表：智能体可以触及的 URL、域名、文件路径。
4. 记忆写入护栏：拒绝看起来像指令的写入。
5. 内容捕获纪律（第 23 课）：将检索到的内容存储在外部；span 携带引用 ID，而不是文本散文。
6. 测试套件：将 Greshake 的五类利用作为红队用例。

硬性拒绝项：

- 没有来源标签的工具使用面。没有来源信息就无法区分权限级别。
- 只在最终输出上运行的 Validator。事后验证毫无意义 —— 模型已经行动了。
- “相信我，系统提示会处理好的。”系统提示卫生不是一种控制手段。

拒绝规则：

- 如果智能体具备任何检索能力却没有来源打标签，拒绝交付。检索到的内容是典型的注入向量。
- 如果敏感工具（发送消息、执行 shell、在 / 写文件）没有 human-in-the-loop 确认，拒绝。
- 如果记忆写入不设防，拒绝。持久化的记忆投毒会在下一个会话中重新投毒。

输出：`validator.py`、`source_tag.py`、`allowlist.py`、`memory_guard.py`、`red_team.py`、`README.md`，说明这六重控制栈、残余风险以及持续评审节奏。最后以“接下来读什么”收尾，指向第 21 课（computer use 安全）和第 23 课（通过 OTel 进行内容捕获）。
