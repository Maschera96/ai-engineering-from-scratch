---
name: migration-agent-zh
description: 构建一个存储库级别的代码迁移代理，将确定性配方与代理回退循环相结合，通过 MigrationBench，并发布故障分类法。
version: 1.0.0
phase: 19
lesson: 09
tags: [capstone, code-migration, openrewrite, libcst, migrationbench, agent, sandbox]
---

给定 Java 8 或 Python 2 存储库，使用绿色测试套件和最小覆盖率回归生成迁移分支（到 Java 17 或 Python 3.12）。对 50 个存储库 MigrationBench 子集进行评估。
建设计划：
1. 确定性传递：OpenRewrite (Java) 或 libcs​​t (Python) 首先运行机械重写。作为具有干净差异的“配方”提交进行提交。
2.Daytona沙箱：预装目标运行时；每个分支的构建；只读源安装。
3. 代理循环：LangGraph 或 OpenAI Agents SDK 通过 Claude Opus 4.7 + GPT-5.4-Codex。工具：<span class="notranslate">`run_build`</span>、<span class="notranslate">`read_file`</span>、<span class="notranslate">`edit_file`</span>、<span class="notranslate">`run_test`</span>、<span class="notranslate">`git_diff`</span>。对失败进行分类（dep、语法、测试、构建工具），应用有针对性的修复，重新运行。
4. 预算上限：30 分钟，8 美元，20 圈。使用当前差异违反 <span class="notranslate">`budget_exhausted`</span> 下的任何停止和文件。
5. 测试+覆盖门：构建绿色然后测试绿色；覆盖率下降不得超过2%。
6. PR 打开配方提交 + 代理提交 + 摘要评论。
7. 失败分类：来自 <span class="notranslate">`{dep_upgrade_required, build_tool_drift, custom_annotation, test_flake, syntax_edge_case, budget_exhausted, coverage_regression}`</span> 的每个存储库标签。
8. 跨 MigrationBench 运行 50 个存储库；发布每个类别的通过率、每个存储库的成本和覆盖率保留；与仅确定性基线进行比较。
评估标准：
|重量 |标准|测量|
|:-:|---|---|
| 25 | 25 MigrationBench 通过率 | 50-repo 子集 pass@1 |
| 20 |测试覆盖率保存 |平均覆盖率增量与基础分支|
| 20 |每个迁移存储库的成本 |平均 $/repo 的传球次数 |
| 20 |代理/确定性工具集成 | OpenRewrite 与代理处理的修复比例 |
| 15 | 15故障分析报告|带有范例的分类完整性 |
硬拒绝：
- 跳过确定性传递的管道。 OpenRewrite 处理机械问题比任何代理便宜 70-80%，而且更可靠。
- 覆盖率回归超过 2% 视为通过。
- 将机械和代理编写的更改捆绑到一次提交中的 PR。必须分开。
- 在相同的 50 个存储库上报告没有匹配的确定性基准的通过率。
拒绝规则：
- 拒绝用力将迁移的树枝推过底座。总是一个新的分支+ PR。
- 拒绝打开 CI 未在沙箱中变绿的 PR。
- 拒绝在没有明确修改许可的情况下在公司存储库上运行。
输出：包含两层迁移管道的存储库、50 个存储库的 MigrationBench 运行日志、故障分类仪表板、匹配的仅确定性基线运行，以及关于三个最常见故障类和消除每个故障类的配方更改的说明。