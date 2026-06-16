# Capstone 09 — 代码迁移代理（回购级语言/运行时升级）
> Amazon 的 MigrationBench（Java 8 到 17）和 Google 的 App Engine Py2-to-Py3 迁移器设定了 2026 年标准。 Moderne 的 OpenRewrite 可以大规模进行确定性 AST 重写。 Grit 的目标是使用 codemod 风格的 DSL 来解决同样的问题。生产模式结合了两者：用于安全重写的确定性基底，用于不明确情况的代理层，用于每个分支构建的沙箱，以及在 PR 打开之前翻转绿色的测试工具。顶峰是迁移 50 个真实的存储库并发布包含失败分类的通过率。
**类型：** 顶点
**语言：** Python（代理）、Java / Python（目标）、TypeScript（仪表板）
**先决条件：** 第 5 阶段（NLP）、第 7 阶段（变压器）、第 11 阶段（LLM 工程）、第 13 阶段（工具）、第 14 阶段（代理）、第 15 阶段（自主）、第 17 阶段（基础设施）
**练习阶段：** P5 · P7 · P11 · P13 · P14 · P15 · P17
**时间：** 30小时
＃＃ 问题
大规模代码迁移是2026编码代理最干净的生产应用之一。事实是显而易见的（迁移后测试套件是否通过？），回报是真实的（Java-8 队列迁移是一个人员规模项目），并且基准是公开的（MigrationBench 50-repo 子集）。 Moderne 的 OpenRewrite 处理确定性方面。代理层可以处理 OpenRewrite 配方无法处理的一切：不明确的重写、构建系统漂移、长尾语法、传递依赖破坏。
您将构建一个代理，该代理采用 Java 8 存储库（或 Python 2 存储库）并生成绿色 CI 迁移分支。您将测量通过率、测试覆盖率保留、每个存储库的成本，并构建失败分类法。与仅确定性基线的并排告诉您代理的价值实际存在于何处。
＃＃ 概念
管道有两层。 **确定性基底**（OpenRewrite for Java，libcs​​t for Python）安全地运行大量机械重写：导入、方法签名、空安全编辑、尝试资源、已弃用的 API 替换。它速度很快并且可以生成可审计的差异。 **代理层**（OpenAI Agents SDK 或 LangGraph 在 Claude Opus 4.7 和 GPT-5.4-Codex）处理配方无法处理的情况：构建文件升级(Maven/Gradle/pyproject)、传递依赖冲突、测试片、自定义注释。
每个存储库都有一个 Daytona 沙箱，其中预装了目标运行时。代理迭代：运行构建、对故障进行分类、应用修复、重新运行。硬限制：每个存储库 30 分钟，每个存储库 8 美元，20 轮代理。如果所有测试都通过并且覆盖范围增量不为负，则分支会打开 PR。如果没有，该存储库将被归档到带有证据的失败类别下。
失败分类是可交付成果。在 50 个存储库中，什么出了问题？传递依赖？自定义注释？构建工具版本？与迁移无关的测试片？每个类都有一个计数和一个示例差异。未来的食谱作者可以瞄准前三名。
＃＃ 建筑学
```
target repo
      |
      v
OpenRewrite / libcst deterministic recipes
   (safe, fast, auditable, ~70-80% of fixes)
      |
      v
Daytona sandbox per branch
      |
      v
agent loop (Claude Opus 4.7 / GPT-5.4-Codex):
   - run build -> capture failures
   - classify failures (build, test, lint)
   - apply fix (patch or retry recipe)
   - rerun
   - budget: 30 min, $8, 20 turns
      |
      v
test + coverage delta gate
      |
      v (passed)
open PR
      |
      v (failed)
file under failure class + attach repro
```

＃＃ 堆
- 确定性基底：OpenRewrite (Java) 或 libcst (Python)
- 代理：OpenAI Agents SDK 或 LangGraph 超过 Claude Opus 4.7 + GPT-5.4-Codex
- 沙箱：每个分支的 Daytona devcontainers，预安装的目标运行时（Java 17 / Python 3.12）
- 构建系统：Maven、Gradle、uv (Python)
- 基准：Amazon MigrationBench 50-repo 子集（Java 8 到 17）、Google App Engine Py2-to-Py3 存储库
- 测试工具：并行运行器，通过 Jacoco (Java) 或coverage.py (Python) 进行覆盖
- 可观察性：Langfuse + 每个存储库的每个差异块的跟踪包
- 仪表板：故障分类仪表板，包含每类计数和示例差异
## 构建它
1. **配方传递。** 首先运行 OpenRewrite (Java) 或 libcs​​t (Python) 配方。捕获 70-80% 的机械迁移。作为“菜谱”提交提交。
2. **构建试用版。** Daytona 沙箱：安装目标运行时，运行构建。如果为绿色，则跳至测试。如果是红色，请交给代理。
3. **代理循环。** LangGraph 使用工具：<span class="notranslate">`run_build`</span>、<span class="notranslate">`read_file`</span>、 <span class="notranslate">`edit_file`</span>、<span class="notranslate">`run_test`</span>、<span class="notranslate">`git_diff`<span类=“notranslate”></span></span>。代理对故障进行分类（dep、语法、测试、构建工具）并应用有针对性的修复。重新运行。
4. **预算上限。** 每个存储库 30 分钟挂钟，成本 8 美元，20 轮代理。任何违规行为都会停止，并根据当前差异将其归档到“budget_exhausted”下。
5. **测试 + 覆盖范围。** 构建变为绿色后，运行测试套件。将覆盖范围与基础存储库进行比较。如果覆盖率下降超过 2%，请归档到“coverage_regression”下。
6. **PR 打开。** 成功后，推送分支，打开 PR，其中包含差异以及应用了哪些配方以及提交了代理所编写的内容的摘要。
7. **失败分类。** 对于每个失败的存储库，使用类标记：<span class="notranslate">`dep_upgrade_required`</span>、<span class="notranslate">`build_tool_drift`</span>、<span class="notranslate">`custom_annotation`</span>、<span class="notranslate">`test_flake`</span>、<span class="notranslate">`syntax_edge_case`</span>、<span class="notranslate">`budget_exhausted`</span>。构建仪表板。
8. **50-repo run。** 在 MigrationBench 子集中执行。报告每类通过率、每个存储库的成本、覆盖率保留以及仅比较与确定性基线。
## 使用它
```
$ migrate legacy-java-service --target java17
[recipe]   27 rewrites applied (JUnit 4->5, HashMap initializer, try-with-resources)
[build]    FAIL: cannot find symbol sun.misc.BASE64Encoder
[agent]    turn 1 classify: removed_jdk_api
[agent]    turn 2 apply: sun.misc.BASE64Encoder -> java.util.Base64
[build]    OK
[tests]    412/412 passing; coverage 84.1% -> 84.3%
[pr]       opened #1841  cost=$3.20  turns=4
```

## 发货
<span class="notranslate">`outputs/skill-migration-agent.md`</span> 是可交付成果。给定一个存储库，它执行确定性配方，然后执行代理循环以生成绿色迁移分支，或将存储库归档到分类​​类下。
|重量 |标准|如何测量 |
|:-:|---|---|
| 25 | 25 MigrationBench 通过率 | 50-repo 子集 pass@1 |
| 20 |测试覆盖率保存 |平均覆盖范围增量与基础 |
| 20 |每个迁移存储库的成本 | $/repo 上的传递运行 |
| 20 |代理/确定性工具集成 | OpenRewrite 处理的修复与代理编写的修复比例
| 15 | 15故障分析报告|带有范例的分类完整性 |
| **100** | | |
## 练习
1. 仅使用 OpenRewrite（无代理）运行迁移管道。将通过率与整个管道进行比较。确定仅由代理人造成差异的情况。
2. 实施“lint-clean”检查：迁移后，运行样式linter（Java 为spotless，Python 为ruff）。如果出现新的 lint 错误，则 PR 失败。测量覆盖率保留但风格回归的比率。
3. 添加“最小差异”优化器：在代理的分支通过测试后，通过第二遍修剪不必要的更改。报告差异大小减少。
4. 扩展到第三次迁移：节点 18 到节点 22。重用沙箱包装；将配方层替换为自定义代码模块。
5. 衡量首次绿色构建时间 (TTFGB) 作为用户体验指标。目标：10 分钟内 p50。
## 关键术语
|术语 |人们怎么说|它实际上意味着什么 ||------|-----------------|------------------------|
|确定性基底| “配方引擎”| OpenRewrite / libcs​​t：具有安全保证的声明式 AST 重写 |
|代码修改| “代码修改程序” |机械更改源代码的重写规则 |
|建立漂移| “工具版本偏差” |主要版本之间 Maven/Gradle/uv 行为的细微变化 |
|故障等级| “分类桶”|存储库未迁移的标记原因：dep、语法、测试、构建工具、预算 |
|覆盖范围增量| “覆盖范围保存” |从基础分支到迁移分支的测试覆盖率变化
|代理转| “工具调用回合”|代理循环中的一个计划 -> 行动 -> 观察循环 |
|预算耗尽| “触及天花板”|该回购协议消耗了 30 分钟/8 美元/20 回合的限制，但没有通过 |
## 进一步阅读
- [Amazon MigrationBench](<span class="notranslate">https://aws.amazon.com/blogs/devops/amazon-introduces-two-benchmark-datasets-for-evaluating-ai-agents-ability-on-code-migration/</span>) — 2026 年规范基准
- [Moderne.io OpenRewrite 平台](<span class="notranslate">https://www.moderne.io</span>) — 确定性基质参考
- [OpenRewrite 文档](<span class="notranslate">https://docs.openrewrite.org</span>) — 配方创作
- [Grit.io](<span class="notranslate">https://www.grit.io</span>) — 备用 codemod DSL
- [OpenAI 沙盒迁移手册](<span class="notranslate">https://developers.openai.com/cookbook/examples/agents_sdk/sandboxed-code-migration/sandboxed_code_migration_agent</span>) — Agents SDK 参考
- [Google App Engine Py2 到 Py3 迁移器](<span class="notranslate">https://cloud.google.com/appengine</span>) — 备用迁移基准
- [libcst](<span class="notranslate">https://github.com/Instagram/LibCST</span>) — Python 确定性基质
- [Daytona 沙箱](<span class="notranslate">https://daytona.io</span>) — 参考每个分支沙箱