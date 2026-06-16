---
name: verification-gate-zh
description: 生成一个确定性的验证关卡，将范围、规则与反馈产物整合为每个任务一份的 verification_report.json，并配套 CI 接线，确保没有绿色裁决就拒绝合并。
version: 1.0.0
phase: 14
lesson: 38
tags: [verification, gate, deterministic, ci, override-log]
---

给定一个项目的验收标准与已有的工作台产物，产出验证关卡与覆盖审计日志。

产出：

1. `tools/verify_agent.py`，对外暴露 `verify(task_id, artifacts) -> VerdictReport`。纯函数，确定性，不调用 LLM。
2. `outputs/verification/<task_id>.json`，作为唯一可信来源的裁决。
3. `tools/override.py`，将签名的覆盖条目追加到 `outputs/verification/overrides.jsonl`（必须包含原因、用户 id、时间戳、发现编码）。
4. CI 工作流，在 `passed: false` 时失败，并就地呈现报告。
5. `docs/verification.md`，列出每一项检查、其严重级别、其来源产物，以及覆盖策略。

硬性拒绝：

- 任何调用 LLM 的检查。关卡是确定性的管道接线；LLM 的判断属于审查者。
- 任何智能体无需签名条目即可走的覆盖路径。覆盖只能由人执行。
- 任何省略其所消费的产物路径的验证报告。报告必须可审计。
- 任何工作流可以悄然降级的 block 级别发现。严重级别在写入时固定，而非在读取时确定。

拒绝规则：

- 如果项目没有验收命令，在其存在之前拒绝交付关卡。一个什么都证明不了的关卡只是作秀。
- 如果规则报告不存在，拒绝跳过规则检查；故障即关闭（fail closed）。
- 如果反馈日志不存在，拒绝跳过验收检查；缺失的日志本身就是一个 block。
- 如果覆盖条目未纳入版本控制，拒绝接入覆盖路径；不留记录的覆盖会让关卡形同虚设。

输出结构：

```
<repo>/
├── tools/
│   ├── verify_agent.py
│   └── override.py
├── outputs/verification/
│   ├── overrides.jsonl
│   └── <task_id>.json
├── docs/verification.md
└── .github/workflows/verify.yml
```

以“接下来读什么”结尾，指向：

- 第 39 课，在绿色裁决之后接手的审查者智能体。
- 第 40 课，将裁决纳入交接包的交接生成器。
- 第 41 课，针对真实风格的示例应用运行该关卡。
