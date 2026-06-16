---
name: permission-mode-picker-zh
description: 为 Claude Code 任务匹配权限模式、预算上限和必要隔离。
版本: 1.0.0
phase: 15
lesson: 10
tags: [claude-code, permission-modes, auto-mode, budgets, isolation]
---

给定a 拟议的 Claude Code 任务, pick the 权限模式, 设置 预算, and 指定 the minimum 隔离 必需 之前 the 智能体 is allowed to start.

产出:

1. **任务 profile.** One sentence on 什么 the 任务 does, 一句话 on the blast radius if it goes wrong.
2. **Mode 建议.** One of: `plan`, `default`, `acceptEdits`, `acceptExec`, `autoMode`, `yolo`, `bypassPermissions`. Justify 带有 a 单一 sentence referencing the blast radius.
3. **预算 numbers.** 具体 values for `max_turns`, `max_budget_usd`, and 任何 per-工具 上限. For unattended runs over an hour, 指定 a dollar 上限 equal to or below 什么 you would pay for a 人类 mistake you 不能 roll back.
4. **隔离 requirements.** File-系统 scope (project directory 只, scratch directory, ephemeral container). 网络 政策 (no egress, allowlist 只, full). 凭据 表面 (none, scoped 词元, broad 词元). For `bypassPermissions` or `yolo`, the 运行 必须 be 内部 an ephemeral container 带有 no 生产环境 凭据 mounted.
5. **轨迹 审计 plan.** 方式 will a 人类审查 the 轨迹 之后 the 运行? 必需 for `autoMode`, `yolo`, and anything over a 30-minute horizon.

硬性拒绝:
- `bypassPermissions` 针对 a repository 带有 uncommitted changes.
- `autoMode` 带有 no 预算 上限.
- 任何 mode above `acceptEdits` 带有 broad 凭据 in the environment (AWS, GCP, GitHub PAT 带有 repo scope).
- Unattended runs longer than one hour 带有 no 轨迹 审计 scheduled.
- Claims that the Auto Mode 分类器 alone is sufficient for a novel 任务 分布.

拒绝规则:
- If the 用户 不能 命名 the blast radius of a 失败, 拒绝 and 要求 an 明确 worst-case sentence 之前 starting.
- If the 用户 requests `autoMode` in a workspace 带有 生产环境 database 凭据 reachable, 拒绝 and 要求 scoped 凭据 or an ephemeral container first.
- If the 拟议的 预算 上限 exceeds 什么 the 用户 is willing to lose on a bad 运行, 拒绝 and 要求 a lower 上限.

输出格式:

返回a one-page 运行 card 带有:
- **任务 摘要** (一句话)
- **Blast radius** (一句话, worst case)
- **Mode** (明确)
- **预算** (`max_turns`, `max_budget_usd`, per-工具 上限)
- **隔离** (fs scope, 网络 政策, 凭据 表面)
- **审计 plan** (人员 reviews the 轨迹, when, 针对 什么 rubric)
