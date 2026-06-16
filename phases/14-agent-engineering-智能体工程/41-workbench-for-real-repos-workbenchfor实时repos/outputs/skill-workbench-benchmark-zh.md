---
name: workbench-benchmark-zh
description: 在项目自带的示例应用上，将同一个任务分别跑通 prompt-only 流水线和 workbench 引导的流水线，并产出包含五种结果的前后对比报告。
version: 1.0.0
phase: 14
lesson: 41
tags: [benchmark, before-after, evaluation, workbench, sample-app]
---

给定一个 repo、一个 agent 产品，以及一个小型示例应用，产出一套可移植的评估框架，用来对比 prompt-only 与 workbench 引导的流水线。

产出物：

1. `eval/sample_app/` —— 从项目所属领域中提炼出的一个最小可行示例应用。
2. `eval/run_prompt_only.py` 和 `eval/run_workbench.py`，二者各自接收一段任务描述并返回一个 `TaskOutcome`。
3. `eval/report.py`，运行两条流水线并写出 `before-after-report.md` 以及 `comparison.json`。
4. 当 workbench 在固定任务套件上的结果出现回退时会失败的 CI 工作流。
5. `docs/benchmark.md`，解释这五种结果，以及什么才算回退。

硬性拒绝项：

- 只有一条流水线的基准测试。对比才是全部意义所在。
- 用百分比表述却没有分母的结果。始终报告 `n / m`。
- 使用 agent 产品训练过的示例应用。请使用针对领域定制的 fixture。
- 隐藏假阴性的报告。prompt-only 更快的任务必须逐一列出。

拒绝规则：

- 如果项目没有验收命令，拒绝交付该基准测试。没有任何可度量的东西。
- 如果在中位任务上 workbench 流水线耗时超过 prompt-only 流水线的 3 倍，请把这一发现暴露出来；需要简化的是 workbench，而不是模型。
- 如果该框架无法离线运行，拒绝把它接入 CI。网络抖动会污染对比结果。

输出结构：

```
<repo>/
├── eval/
│   ├── sample_app/
│   ├── run_prompt_only.py
│   ├── run_workbench.py
│   └── report.py
├── outputs/eval/
│   ├── before-after-report.md
│   └── comparison.json
├── docs/benchmark.md
└── .github/workflows/benchmark.yml
```

以“接下来读什么”收尾，指向：

- 第 42 课，关于将 workbench 流水线用到的每个面打包在一起的收官合集。
- 第 19 课（SWE-bench、GAIA、AgentBench），与之互补的宏观基准测试。
- 第 30 课（Eval-Driven Agent Development），关于基准测试接好之后持续运行的评估循环。
