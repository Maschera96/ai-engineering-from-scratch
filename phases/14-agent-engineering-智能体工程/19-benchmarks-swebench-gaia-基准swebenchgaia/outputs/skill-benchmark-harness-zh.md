---
name: benchmark-harness-zh
description: 为代码库构建一个 SWE-bench 风格的测试框架，带有 FAIL_TO_PASS / PASS_TO_PASS 门控、污染检查和步数指标。
version: 1.0.0
phase: 14
lesson: 19
tags: [swe-bench, gaia, agentbench, harness, evaluation]
---

给定一个代码库和一组 (bug, fix) 对，构建一个基准测试框架，以真实的单元测试作为门控，并记录运行指标。

产出：

1. 每个任务的定义：`(tid, description, state_before, fail_to_pass_tests, pass_to_pass_tests, solution)`。
2. 一个运行器，应用智能体的补丁，在沙箱中运行仓库的测试套件，并记录：FTP 通过数、PTP 通过数、步数、tokens、墙钟时间、成本。
3. 一个污染检查：将 issue 文本与生成的补丁做模式匹配；标记 >=30% 的重叠。
4. 一个报告器，将每个任务的分数和聚合分数以 JSON 形式输出，外加 P50/P75/P95 的步数和成本。
5. 一个 CI 作业，在每个 PR 上运行该框架，并在 >=5% 回归时失败。

硬性拒绝：

- 只报告单一聚合数字的框架。要求提供每个任务的结果 + 分布。
- 不在沙箱中运行测试的框架。智能体提供的补丁是不受信任的代码。
- 没有 PASS_TO_PASS 门控的框架。破坏其他测试的补丁会悄悄使产品回归。

拒绝规则：

- 如果用户要求“只要 FAIL_TO_PASS 分数”，拒绝。要加上 PASS_TO_PASS；破坏现有测试比没修好问题是更严重的回归。
- 如果测试没有固定到某个具体 commit，拒绝。测试漂移会使各次运行之间的分数无法比较。
- 如果任务与训练期间见过的 issue 文本有重叠，明确标记出来。

输出：`tasks.py`、`harness.py`、`contamination.py`、`report.py`、`README.md`，解释沙箱、门控、污染策略。结尾以 “接下来读什么” 指向第 30 课，讲解在该框架之上进行评估驱动的开发。
