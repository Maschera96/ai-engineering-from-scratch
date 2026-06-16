# Capstone 10 — 多代理软件工程团队
> SWE-AF 的工厂架构、MetaGPT 的基于角色的提示、AutoGen 0.4 的类型化演员图、Cognition 的 Devin 和 Factory 的 Droid 都汇聚在同一个 2026 年形状上：架构师计划、N 个编码员在并行工作树中工作、审阅者门控、测试人员验证。并行工作树将时钟转换为吞吐量。共享状态和切换协议成为故障面。顶峰是建立团队，在 SWE-bench Pro 上进行评估，并报告哪些切换中断以及频率如何。
**类型：** 顶点
**语言：** Python / TypeScript（代理）、Shell（工作树脚本）
**先决条件：** 第 11 阶段（LLM 工程）、第 13 阶段（工具）、第 14 阶段（代理）、第 15 阶段（自主）、第 16 阶段（多代理）、第 17 阶段（基础设施）
**练习的阶段：** P11 · P13 · P14 · P15 · P16 · P17
**时间：** 40小时
＃＃ 问题
单代理编码工具在大型任务上达到了上限。不是因为任何单个代理都很弱，而是因为 200k 令牌上下文无法容纳架构计划加上四个并行代码库切片加上审阅者评论加上测试输出。多代理工厂解决了问题：架构师拥有计划，编码员自己在并行工作树中实现，审查者控制，测试人员验证。 SWE-AF 的“工厂”架构、MetaGPT 的角色、AutoGen 的类型化参与者图 — 所有三个框架都描述相同的形状。
故障面是切换。架构师计划了一些编码员无法实现的事情。编码员会产生相互冲突的差异。审阅者批准了幻觉修复。测试人员与仍在编写代码的编码人员竞争。您将建立其中一个团队，在 50 个 SWE-bench Pro 问题上运行它，跟踪每次交接，并发布事后分析。
＃＃ 概念
角色是类型化的代理。 **架构师** (Claude Opus 4.7) 读取问题，编写计划，并将其分解为具有显式接口的子任务。 **编码员**（Claude Sonnet 4.7，N 个并行实例，每个实例位于 <span class="notranslate">`git worktree`</span> + Daytona 沙箱中）独立实现子任务。 **审阅者** (GPT-5.4) 读取合并的差异并批准或请求特定更改。 **测试人员** (Gemini 2.5 Pro) 独立运行测试套件并报告 pass/fail 以及工件。
通过共享任务板（文件支持或 Redis）进行通信。每个角色都会消耗它被允许处理的任务。切换是 A2A 协议类型的消息。协调问题：合并冲突解决（协调员角色或自动三向合并）、共享状态同步（编码员开始后计划被冻结；重新计划是单独的事件）和审阅者把关（审阅者不能批准自己的更改或它提议的更改）。
代币放大是隐性成本。每个角色边界都会添加摘要提示和切换上下文。 40 个回合的单代理运行变成了 4 个角色总共 160 个回合。该标题特别权衡了代币效率与单代理基线，因为问题不是“多代理是否有效”，而是“它是否能赢得每一美元”。
＃＃ 建筑学
```
GitHub issue URL
      |
      v
Architect (Opus 4.7)
   reads issue, produces plan with subtasks + interfaces
      |
      v
Task board (file / Redis)
      |
   +-- subtask 1 ---+-- subtask 2 ---+-- subtask 3 ---+-- subtask 4 ---+
   v                v                v                v                v
Coder A          Coder B          Coder C          Coder D          (4 parallel)
 (Sonnet)         (Sonnet)         (Sonnet)         (Sonnet)
 worktree A       worktree B       worktree C       worktree D
 Daytona          Daytona          Daytona          Daytona
      |                |                |                |
      +--------+-------+-------+--------+
               v
           merge coordinator  (three-way merge + conflict resolution)
               |
               v
           Reviewer (GPT-5.4)
               |
               v
           Tester  (Gemini 2.5 Pro)  -> passes? -> open PR
                                     -> fails?  -> route back to coder
```

＃＃ 堆
- 编排：LangGraph 具有共享状态+每个代理子图
- 消息传递：用于键入代理间消息的 A2A 协议 (Google 2025)
- 型号：Opus 4.7（架构师）、Sonnet 4.7（编码员）、GPT-5.4（审阅者）、Gemini 2.5 Pro（测试员）
- 工作树隔离：每个编码器 <span class="notranslate">`git worktree add`</span> + Daytona 沙箱
- 合并协调员：自定义三向合并+LLM介导的冲突解决
- 评估：SWE-bench Pro（50 个问题）、SWE-AF 场景、用于单元测试的 HumanEval++
- 可观察性：Langfuse 带有角色标记的跨度，每个代理令牌会计
- 部署：K8s，每个角色作为单独的部署 + 积压的 HPA
## 构建它
1. **任务板。** 带键入消息的文件支持 JSONL：<span class="notranslate">`plan_request`</span>、<span class="notranslate">`subtask`</span>、<span class="notranslate">`diff_ready`</span>、<span class="notranslate">`review_needed`</span>、<span class="notranslate">`test_needed`</span>、<span class="notranslate">`approved`</span>、<span class="notranslate">`rejected`</span>、<span class="notranslate">`replan_needed`</span>。代理订阅标签。
2. **架构师。** 阅读 GitHub 问题，使用需要显式子任务接口（涉及的文件、公共函数、测试影响）的计划模板运行 Opus 4.7。发出一个 <span class="notranslate">`plan_request`</span> 以及子任务的 DAG。
3. **编码员。** N 个并行工作人员，每个工作人员从董事会领取一项子任务。每个分支都会生成一个新的 <span class="notranslate">`git worktree add`</span> 分支以及 Daytona 沙箱。实现子任务。发出 <span class="notranslate">`diff_ready`</span> 以及补丁 + 测试增量。
4. **合并协调器。** 在所有编码器完成后，三路将 N 个分支合并到一个暂存分支中。仅当存在文件级重叠时，LLM 介导的冲突解决。
5. **审阅者。** GPT-5.4 读取合并的差异。无法批准其创作的差异。发出 <span class="notranslate">`approved`</span> （无操作）或 <span class="notranslate">`review_feedback`</span> ，并将特定更改请求路由回相关编码器。
6. **测试器。** Gemini 2.5 Pro 在干净的沙箱中运行测试套件。捕获文物。发出 <span class="notranslate">`test_passed`</span> 或 <span class="notranslate">`test_failed`</span> 以及堆栈跟踪。失败的测试会循环回拥有失败子任务的编码人员。
7. **切换记帐。** 每条跨越角色边界的消息都会在 Langfuse 中获得一个跨度，其中包含有效负载大小和使用的模型。计算每个子任务令牌放大（coder_tokens + reviewer_tokens + tester_tokens + archite_share / coder_tokens）。
8. **评估。** 在 50 个 SWE-bench Pro 问题上运行。将 pass@1 和 $-per-solved-issue 与单代理基线（单个工作树中的一个 Sonnet 4.7）进行比较。
9. **事后分析** 对于每个失败的问题，确定中断的交接（计划太模糊、合并冲突、审阅者错误批准、测试人员分裂）。生成切换失败直方图。
## 使用它
```
$ team run --issue https://github.com/acme/widget/issues/842
[architect] plan: 4 subtasks (parser, cache, api, migration)
[board]     dispatched to 4 coders in parallel worktrees
[coder-A]   subtask parser  -> 42 lines, tests pass locally
[coder-B]   subtask cache   -> 88 lines, tests pass locally
[coder-C]   subtask api     -> 31 lines, tests pass locally
[coder-D]   subtask migration -> 19 lines, tests pass locally
[merge]     3-way merge: 0 conflicts
[reviewer]  comments on cache (thread pool sizing); routed to coder-B
[coder-B]   revision: 92 lines; submits
[reviewer]  approved
[tester]    all 412 tests pass
[pr]        opened #3382   4 coders, 1 revision, $4.90, 18m
```

## 发货
<span class="notranslate">`outputs/skill-多智能体-team.md`</span> 是可交付成果。给定问题 URL 和并行级别，团队会生成一个可合并的 PR，并按角色进行令牌记账。
|重量 |标准|如何测量 |
|:-:|---|---|
| 25 | 25 SWE-bench Pro pass@1 |匹配的 50 个问题子集，pass@1 |
| 20 |并行加速 |挂钟与单代理基线 |
| 20 |审查质量 |注入错误探测器的错误批准率
| 20 |代币效率 |每个已解决问题的总代币与单一代理的比较 |
| 15 | 15协调工程|合并冲突解决、切换失败直方图 |
| **100** | | |
## 练习
1. 在运行中的 diff 中注入一个明显的错误（在主体之前额外的 <span class="notranslate">`return None`</span>）。衡量审稿人的错误批准率。调整审阅者提示，直到错误批准率低于 5%。
2. 减少到两个编码员（架构师 + 编码员 + 评审员 + 测试员，编码员顺序运行两个子任务）。比较挂钟和通过率。
3. 将合并协调器替换为单写入器约束（子任务涉及不相交的文件集）。衡量建筑师的规划负担。
4. 将审阅者从 GPT-5.4 替换为 Claude Opus 4.7。衡量错误批准率和代币成本增量。
5. 添加第五个角色：记录员（Haiku 4.5）。审核后，它会生成一个变更日志条目。衡量文档质量是否值得额外的代币支出。
## 关键术语
|术语 |人们怎么说|它实际上意味着什么 ||------|-----------------|------------------------|
|并行工作树| “孤立的分支”| <span class="notranslate">`git worktree add`</span> 为每个编码器生成一个新的工作树 |
|任务板| “共享消息总线” |代理订阅的键入消息的文件或 Redis 存储 |
|交接 | “角色边界” |任何从一个角色的上下文跨越到另一个角色的上下文的消息 |
|代币放大 | “多代理开销” |同一任务的跨角色/单代理令牌总数 |
| A2A 协议 | “代理对代理”| Google 的 2025 年打字代理间消息规范 |
|合并协调员| “积分器” |运行三向合并并调解冲突的组件 |
|虚假批准| “审稿人幻觉”|审阅者批准了与已知错误的差异 |
## 进一步阅读
- [SWE-AF工厂架构](<span class="notranslate">https://github.com/Agent-Field/SWE-AF</span>) — 参考2026年多代理工厂
- [MetaGPT](<span class="notranslate">https://github.com/FoundationAgents/MetaGPT</span>) — 基于角色的多代理框架
- [AutoGen v0.4](<span class="notranslate">https://github.com/microsoft/autogen</span>) — Microsoft 的类型化 Actor 框架
- [认知 AI (Devin)](<span class="notranslate">https://cognition.ai</span>) — 参考产品
- [Factory Droids](<span class="notranslate">https://www.factory.ai</span>) — 替代参考产品
- [Google A2A 协议](<span class="notranslate">https://developers.google.com/agent-to-agent</span>) — 代理间消息传递规范
- [git worktree 文档](<span class="notranslate">https://git-scm.com/docs/git-worktree</span>) — 隔离基板
- [SWE-bench Pro](<span class="notranslate">https://www.swebench.com</span>) — 评估目标