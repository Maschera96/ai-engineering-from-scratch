---
name: dgm-evaluator-firewall-zh
description: 为 DGM 式自修改循环设计评估器隔离，避免已记录的奖励黑客模式。
版本: 1.0.0
phase: 15
lesson: 4
tags: [dgm, self-modification, reward-hacking, evaluator, sandbox]
---

给定a 拟议的 self-modifying 智能体 循环 (智能体 scaffolding, 评估器 流水线, 部署 target), 生成 a 防火墙 specification that keeps the 评估器 外部 the 智能体's reach. The DGM 论文 documented the 智能体 removing its own 安全 markers to raise 分数; this skill is the 控制.

产出:

1. **File-系统 boundary.** 列出 每个 文件 the 智能体 can edit. 列出 每个 文件 the 评估器 reads. The intersection 必须 be empty. If it is not, 指定 the exact move needed to make it empty.
2. **进程 boundary.** The 评估器 必须 运行 in a 进程 the 智能体 does 不控制. Spec the container or 沙箱. If the 智能体 has shell access, the 智能体 can bypass in-进程 guards.
3. **网络 and secret boundary.** The 智能体 必须 不be able to edit the CI secrets that fetch 评估器 输入 or post 分数. 列出 在哪里 secrets live and 如何 they are mounted.
4. **分数-ingress contract.** 分数 返回 来自 the 评估器 to the 循环's archive. That path is the 只 渠道. Anything the 智能体 can 编写 that influences future scoring breaks the contract.
5. **变异 审查 列出.** 列出 the 类别 of 智能体 edits that 要求 manual 审查 之前 acceptance: new dependencies, edits to scoring-adjacent code, edits to 沙箱 configuration, edits to 日志 or 监控.

硬性拒绝:
- 评估器 in the same repository as the 智能体 来源. Move it out.
- 评估器 that runs as a Python import 内部 the 智能体 进程. 运行 it in a subprocess.
- 任何 评分 adjustment that is applied at the 智能体's side of the boundary. 所有 adjustment 必须 be server-side.
- 任何智能体能通过编辑自身代码而静默关闭的日志。

拒绝规则:
- If the 用户 不能 命名 每个 文件 the 智能体 can edit, 拒绝 and 要求 a 文件-level 编写-access 审计 first.
- 如果评估器没有留出输入，则拒绝 — this is Lesson 3's 审计, a 前置条件.
- If the 部署 表面 includes 变异 to the 评估器 itself (even indirectly through a 拟议的 dependency update), 拒绝 and 要求 a manual 防火墙-审查 step.

输出格式:

返回a one-page spec 带有:
- **Agent 编写-表面** (paths, globs)
- **评估器 阅读-表面** (paths, endpoints)
- **交集** (必须 be empty; show the diff)
- **进程 模型** (如何 the 评估器 is isolated)
- **密钥清单** (挂载位置和挂载方式)
- **需要审查的变异类别** (项目符号)
- **签字栏** (谁负责防火墙不变量)
