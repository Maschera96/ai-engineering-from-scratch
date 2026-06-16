---
name: feedback-runner-zh
description: 用确定性的 stdout/stderr/退出码/耗时捕获包装 shell 命令，为每条命令持久化一条 JSONL 记录，并在反馈缺失时拒绝推进智能体循环。
version: 1.0.0
phase: 14
lesson: 37
tags: [feedback, subprocess, runner, jsonl, loop-control]
---

给定一个在智能体循环内运行 shell 命令的项目，产出一个反馈 runner 以及它写入的 JSONL。

产出：

1. `tools/run_with_feedback.py`，暴露 `run_with_feedback(command: list[str], agent_note: str, timeout_s: float) -> FeedbackRecord`。
2. workbench 下的 `feedback_record.jsonl` 位置，每条命令一行记录。
3. `tools/feedback_loader.py`，返回当前活动任务的最近 N 条记录。
4. 一个 `loop_can_advance(record) -> bool` 辅助函数，智能体循环在宣称成功前调用它。
5. 覆盖以下场景的测试：成功路径、非零退出、超时、缺失二进制文件、确定性的头/尾截断。

硬性拒绝项：

- runner 中任何地方使用 `shell=True`。仅允许 argv。
- 依赖墙上时钟或随机采样的截断。相同输入必须产出相同记录。
- 缺少 `duration_ms` 的记录。探针变慢是 workbench 卡死的第一个征兆。
- 返回无界列表的 loader。要限制在最后 N 条或分页。

拒绝规则：

- 如果项目通过 stdout 传递机密，拒绝在没有脱敏步骤的情况下交付 runner。把本会被捕获的那些行暴露出来。
- 如果项目存在可能无限挂起的命令，拒绝在没有默认超时和显式覆盖列表的情况下交付。
- 如果 runner 运行在带共享状态的 worker 内，拒绝跳过对 JSONL 追加的文件锁。多个写入者会撕裂文件。

输出结构：

```
<repo>/
├── feedback_record.jsonl
└── tools/
    ├── run_with_feedback.py
    ├── feedback_loader.py
    └── test_feedback_runner.py
```

以"接下来读什么"结尾，指向：

- 第 38 课，消费这些记录的验证门控。
- 第 39 课，在为一次运行评分时读取反馈的审阅者智能体。
- 第 23 课，在反馈稳固后添加到遥测侧的 OTel GenAI 约定。
