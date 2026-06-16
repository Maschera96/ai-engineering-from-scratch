---
name: issue-to-pr-zh
description: 构建一个异步 GitHub issue-to-PR agent，在云沙箱中运行、复现 build、验证 tests，并在严格 per-repo budgets 内打开 review-ready PRs。
version: 1.0.0
phase: 19
lesson: 16
tags: [capstone, async-agent, github, fargate, daytona, swe-bench, budget, safety]
---

给定一个带 `@agent fix this` 标签的 GitHub repository，交付一个自托管 cloud agent：把每个带标签 issue 转成 review-ready PR，凭据受限，成本有界。

构建计划：

1. GitHub App 使用 fine-grained token：issues rw、PRs write、contents rw、workflows read。禁止 force-push。main 上 branch protection 阻止直接写入。
2. Webhook receiver（Lambda 或 Fly.io）过滤 label / PR-comment events，并入队到 SQS。
3. Dispatcher 强制 per-repo per-day 美元和 PR 数上限；每个允许 job 启动一个 ECS Fargate task。
4. Environment inference：从 repo contents 检测 language + package manager + runtime。如果缺失 Dockerfile，则即时合成。
5. 每个 task 使用 Daytona 或 E2B sandbox。把 repo clone 到新的 `git worktree` + agent branch。
6. Agent loop（mini-swe-agent 或 SWE-agent v2，模型为 Claude Opus 4.7 或 GPT-5.4-Codex）。工具：ripgrep、tree-sitter repo-map、read_file、edit_file、run_tests、git。Caps：20 美元、30 turns、30 分钟。
7. Verify：sandbox 内完整 CI；用 jacoco / coverage.py 计算 coverage delta；若 delta < -2% 标记 `needs-review`；CI red 则 halt。
8. 通过 GitHub API 打开 PR，包含 rationale、diff summary、trace URL、cost、turns。
9. Observability：每个 PR 一个 Langfuse trace；log scrub secrets；per-repo budget dashboard。
10. 在 30 个 seeded internal issues 上 eval；在三个共享 issues subset 上与 Cursor Background Agents 和 AWS Remote SWE Agents 对比。

评估标准：

| Weight | Criterion | Measurement |
|:-:|---|---|
| 25 | 30 issues pass rate | 端到端成功（CI green + coverage OK） |
| 20 | PR 质量 | Diff size、coverage delta、style conformance |
| 20 | 每个 resolved issue 的成本和延迟 | $/PR 和 wall-clock/PR |
| 20 | 安全 | Scoped token、per-repo budget、no force-push、credential hygiene |
| 15 | Operator UX | Rationale comments、retry affordance、@-mention follow-up |

硬性拒收：

- 任何可以 force-push 的 agent。硬性排除。
- 跳过 budget checks 的 dispatchers。Runaway loops 是经典失败。
- 未通过 sandbox 中完整 CI 就打开 PR。
- Trace archives 包含未脱敏 tokens 或 PII。

拒绝规则：

- main 未启用 branch protection 时，拒绝安装。
- 没有 per-repo daily budget（dollars 和 PR count）时，拒绝运行。
- 拒绝自动 retry failed runs；所有 retries 都需要人工重新添加 label。

输出：一个仓库，包含 GitHub App、webhook receiver、dispatcher + budget ledger、Fargate task definition、sandbox lifecycle manager、mini-swe-agent loop、30-issue eval run、与 Cursor Background Agents 和 AWS Remote SWE Agents 的 side-by-side comparison，以及一份说明文档，列出前三类 build-inference failures 和降低每类问题的 Dockerfile-synthesis change。
