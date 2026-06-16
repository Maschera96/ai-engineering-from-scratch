---
name: aar-deployment-review-zh
description: 在自动化对齐研究流水线部署前审查沙箱、论坛日志、任务分配和人类门禁。
版本: 1.0.0
phase: 15
lesson: 6
tags: [aar, alignment-research, sandbox, log-integrity, rsp]
---

给定a 拟议的 Automated-对齐-Research 部署 (模型, 沙箱, 任务 queue, 论坛), 生成 a pre-部署 审查 that a frontier-lab 安全 team would sign off on.

产出:

1. **沙箱 隔离.** 验证 文件系统, 网络, and 进程 限制 for each AAR instance. 确认 no AAR can 阅读 or modify another AAR's 沙箱. 确认 no AAR can reach the 论坛 存储's 编写 endpoint 直接 (所有 posts 通过 through a separate service).
2. **论坛 日志 integrity.** The 论坛 必须 be 只追加 and 可证明篡改的. 指定 the 存储 (e.g. S3 带有 object lock, immutable ledger, 只追加 WAL). 指定 the verification interval and the response if tampering is detected.
3. **任务 分配 政策.** 说明 如何 任务 are assigned: 人员 writes them, 人员 approves them, 是否 AARs can propose their own follow-up 任务, and under 什么 conditions. Free decomposition wins on 性能; prescribed 工作流 wins on auditability. Document the tradeoff explicitly.
4. **人类 审查 gate.** 之前 任何 AAR 输出 influences real 研究 direction, training data, or published work, a 具名 人类审查er 必须 sign off. 指定 the 审查 checklist: experiment reproducibility, 日志 completeness, 声明-to-证据 mapping.
5. **阈值 监控.** 跟踪 signals that the AAR 系统 approaches RSP v3.0 AI R&D-4 territory: 任务 complexity delta, 自主 sub-任务 spawning, cross-任务 information transfer. 定义 阈值 that 要求 elevated 审查.

硬性拒绝:
- 任何 AAR 部署 没有 可证明篡改的, out-of-沙箱 日志.
- 任何 流水线 在哪里 AARs' 输出 feed 直接 进入 training data or 生产环境 政策 没有 具名 人类审查.
- 任何 流水线 在哪里 a 单一 AAR has sufficient 凭据 to influence multiple downstream systems.

拒绝规则:
- If the 沙箱 隔离 is unspecified or relies on a 单一 层 (Docker 只, no seccomp / gVisor), 拒绝 and 要求 防御-in-depth.
- If the 日志 存储 is editable by anyone (even operators), 拒绝 and 要求 编写-once media.
- If the 部署's goal is to automate a part of the 能力 流水线 — 不just 对齐 研究 — 拒绝 and escalate to RSP 审查.

输出格式:

返回a 审查 memo 带有:
- **流水线 摘要** (一段话)
- **隔离 评分** (per-dimension: fs, net, proc, peer)
- **Log integrity 评分** (带有 verification plan)
- **任务 分配 decision** (fixed / free / hybrid, 带有 理由)
- **人类 审查 gate** (审查者 命名, checklist)
- **阈值 monitors** (列出 of signals, 阈值, response)
- **部署 结论** (通过 / 暂停 / 不通过)
