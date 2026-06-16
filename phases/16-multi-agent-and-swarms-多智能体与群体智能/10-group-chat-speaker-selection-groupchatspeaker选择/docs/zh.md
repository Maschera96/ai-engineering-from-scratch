# 群聊和发言人选择
> AutoGen GroupChat 和 AG2 GroupChat 在 N 个座席之间共享一个对话；选择器功能（LLM、循环法或自定义）选择下一个发言者。这是紧急多代理对话的原型——代理不知道自己在静态图中的角色，他们只是对共享池做出反应。 AutoGen v0.2 的 GroupChat 语义保留在 AG2 分支中； AutoGen v0.4 将其重写为事件驱动的 Actor 模型。 Microsoft 于 2026 年 2 月将 AutoGen 置于维护模式，并将其与语义内核合并到 Microsoft Agent Framework（RC 2026 年 2 月）。 GroupChat 原语在 AG2 和 Microsoft Agent Framework 中都存在 — 学习一次，随处使用。
**类型：** 学习 + 构建
**语言：** Python (stdlib)
**先决条件：** 第 16 阶段·04（原始模型）
**时间：** ~60 分钟
＃＃ 问题
当工作流程已知时，静态图 (LangGraph) 非常有用。真正的对话不是静态的：有时程序员询问审阅者，有时询问研究人员，有时询问作者。对每个可能的切换进行硬编码会产生边缘爆炸。您希望*代理对共享池做出反应*，并通过某些功能来决定接下来由谁发言。
这正是 AutoGen GroupChat 所做的。
＃＃ 概念
### 形状
```
              ┌─── shared pool ────┐
              │   m1  m2  m3  ...  │
              └─────────┬──────────┘
                        │ (everyone reads all)
      ┌───────┬─────────┼─────────┬───────┐
      ▼       ▼         ▼         ▼       ▼
    Agent A  Agent B  Agent C  Agent D  Selector
                                           │
                                           ▼
                                  "next speaker = C"
```

每个代理都会看到每条消息。每次都会调用选择器函数来选择下一个发言者。
### 三种选择器风格
**循环赛。**固定周期。确定性的。在 N 中线性扩展，但忽略上下文——即使主题是法律审查，编码员也能得到轮流。
**LLM-selected。** 对 LLM 的调用，读取最近的池并返回最佳的下一个演讲者。上下文感知但速度慢：每个回合都会添加一个 LLM 调用。 AutoGen 的默认值。
**自定义。** 具有您想要的任何逻辑的 Python 函数。典型：法学硕士选择并具有后备规则（例如，“始终在编码器之后轮到验证者”）。
### ConversableAgent API
```
agent = ConversableAgent(
    name="coder",
    system_message="You write Python.",
    llm_config={...},
)
chat = GroupChat(agents=[coder, reviewer, tester], messages=[])
manager = GroupChatManager(groupchat=chat, llm_config={...})
```

`GroupChatManager` 保存选择器。当代理完成一轮时，管理器调用选择器，该选择器返回下一个代理。循环继续直至出现终止条件。
### 终止
三种常见模式：
- **最大轮数。** 总轮数的硬性上限。
- **“TERMINATE”令牌。** 代理可以发出哨兵消息；当有人出现时，经理停下来。
- **目标达成检查。** 轻量级验证程序每轮运行并在完成后停止聊天。
### AutoGen → AG2 拆分和 Microsoft Agent Framework 合并
2025 年初，微软开始围绕事件驱动的 Actor 模型对 AutoGen (v0.4) 进行重大重写。社区将 AutoGen v0.2 的 GroupChat 语义分叉为 AG2，保留了早期采用者集成的 API。
2026 年 2 月，微软宣布 AutoGen 将进入维护模式，事件驱动的参与者模型将合并到 **Microsoft Agent Framework**（RC 2026 年 2 月，现已与语义内核合并）。 GroupChat 的概念在两条轨道上都存在。实施细节有所不同。 AG2 是 v0.2 兼容代码的首选上游。
### 当 GroupChat 适合时
- **紧急对话。** 您不想预先连接每个可能的下一个发言者。
- **角色混合任务。** 编码员询问研究员，研究员询问档案管理员，档案管理员又询问编码员。流不是 DAG。
- **探索性解决问题。** 想想“头脑风暴会议”，而不是“装配线”。
### 当失败时
- **严格的确定性。** LLM 选择器可能不一致。相同的提示，不同的运行，不同的下一个发言者。
- **阿谀奉承。** 特工们会听从最自信发言者的意见。明确反提示。
- **上下文膨胀。** 每个代理都会读取每条消息； 10回合后，背景就变得巨大了。使用投影（第 15 课）来确定视图范围。
- **热门演讲者。** 一名代理主导了对话，因为选择者偏爱其专业。引入扬声器平衡作为选择器功能。
### 群聊 vs 主管
相同的原语，不同的默认值：
- 主管：一名代理人制定计划，其他代理人执行。选择器是“询问计划者要做什么”。
- 群聊：所有座席都是同行；选择器是共享池上的一个函数。
两者都使用第 04 课中的四个原语。群聊默认为 LLM 选择的编排和全池共享状态。
## 构建它
`code/main.py` 在 stdlib 中从头开始实现 GroupChat。三个代理（编码员、审阅者、经理）、循环法和 LLM 选择的变体，以及 `TERMINATE` 令牌上的终止。
该演示打印了对话记录以及两个变体的选择者的决策跟踪。
跑步：
```
python3 code/main.py
```

## 使用它
`outputs/skill-groupchat-selector.md` 为给定任务配置 GroupChat 选择器 — 循环、LLM 选择、自定义，以及要使用的选择器输入（最近消息、代理专业、回合计数）。
## 发货
清单：
- **最大回合上限。** 始终如此。典型任务为 10-20。
- **扬声器平衡指标。** 跟踪每个座席的轮次；当不平衡超过阈值时发出警报。
- **终止令牌。** `TERMINATE` 或专用验证代理。
- **投影或范围内存。** 在大约 10 条消息之后，考虑只为每个代理提供一个范围视图，以防止上下文膨胀。
- **选择器日志记录。** 对于 LLM 选择的变体，记录选择器的输入及其选择。否则无法调试。
## 练习
1.运行`code/main.py`。比较循环赛和法学硕士选择下的对话。每个代理下哪个代理占主导地位？
2. 在选择器中添加“max-speaks-per-agent”规则。它如何影响转录？
3. 实施达到目标的终止：当审核者返回“已批准”时停止。在圆帽之前触发的频率是多少？
4. 阅读 GroupChat (https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/design-patterns/group-chat.html) 上的 AutoGen 稳定文档。确定 `GroupChatManager` 使用的默认选择器。
5. 阅读 AG2 存储库 (https://github.com/ag2ai/ag2) 并将其 v0.2 GroupChat 与 v0.4 事件驱动版本进行比较。 v0.4 添加了哪些具体属性（吞吐量、容错性、可组合性）？
## 关键术语
|术语 |人们怎么说|它实际上意味着什么 ||------|----------------|------------------------|
|群聊 | “一个聊天室中的代理”|共享消息池+选择器功能。 AutoGen / AG2 原语。 |
|音箱选型| “下一个发言者是谁”|选择下一个代理的函数。循环赛、LLM 选择或自定义。 |
|群聊管理器 | “会议主持人”| AutoGen 组件拥有选择器并循环循环。 |
|对话代理 | “基地特工”| AutoGen 基类；可以发送和接收消息的代理。 |
|终止令牌 | “‘停止’这个词”|结束聊天的哨兵字符串（通常是 `TERMINATE`）。 |
|热门音箱| “一个代理称霸”|选择器不断选择相同代理的失败模式。 |
|上下文膨胀 | “泳池无限增长”|每个代理都会读取之前的每条消息；上下文随着轮流而增长。 |
|投影| “范围视图” |共享池中特定于角色的视图可防止上下文膨胀。 |
## 进一步阅读
- [AutoGen 群聊文档](https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/design-patterns/group-chat.html) — 参考实现
- [AG2 repo](https://github.com/ag2ai/ag2) — 社区 AutoGen v0.2 延续
- [Microsoft Agent Framework 文档](https://microsoft.github.io/agent-framework/) — 合并的后继者，RC 2026 年 2 月
- [AutoGen v0.4 发行说明](https://microsoft.github.io/autogen/stable/) — 事件驱动的 Actor 模型重写详细信息