# Capstone 16 — GitHub Issue 到 PR 的自主代理

> AWS Remote SWE Agents、Cursor Background Agents、OpenAI Codex cloud 和 Google Jules，都在 2026 年推出了同一种产品形态，给 issue 打个标签，就能得到一个 PR。Agent 在云端沙箱里运行，通过测试验证，再提交一个带 rationale 的 review-ready PR。难点在于如何自动复现实验仓库的构建环境、避免凭据泄漏、对每个仓库施加预算约束，以及确保 agent 不能 force-push。这个 capstone 要做一个自托管版本，并在成本与通过率上与托管替代方案对比。

**Type:** Capstone
**Languages:** Python (agent), TypeScript (GitHub App), YAML (Actions)
**Prerequisites:** Phase 11 (LLM engineering), Phase 13 (tools), Phase 14 (agents), Phase 15 (autonomous), Phase 17 (infrastructure)
**Phases exercised:** P11 · P13 · P14 · P15 · P17
**Time:** 30 hours

## Problem

异步云端编码 agent 与交互式编码 agent，也就是 capstone 01，是不同的产品类别。它的 UX 就是一个 GitHub label。你给 issue 打上 `@agent fix this` 标签，一个 worker 就会在云端沙箱里启动，克隆 repo，运行测试，编辑文件，做验证，并创建一个 PR，把 agent 的理由写进正文。没有交互式循环，也没有终端。AWS Remote SWE Agents、Cursor Background Agents、OpenAI Codex cloud、Google Jules 和 Factory Droids 都在往这个方向收敛。

工程难点非常具体，环境复现，agent 必须在没有缓存 dev image 的情况下从零构建 repo，测试不稳定时要重跑或隔离，凭据范围控制，要靠最小权限的 GitHub App，预算控制，要做到每仓库每天有上限，还要确保 no-force-push policy。这个 capstone 要测量 pass rate、cost 和 safety，并与托管替代方案对比。

## Concept

触发器是 GitHub webhook，可能来自 issue label，也可能来自 PR comment。Dispatcher 把工作放进 ECS Fargate 或 Lambda。Worker 把 repo 拉进 Daytona 或 E2B sandbox，并根据 repo 的语言与框架推断出一个通用 Dockerfile。然后 agent 以 mini-swe-agent 或 SWE-agent v2 loop 的方式，在 Claude Opus 4.7 或 GPT-5.4-Codex 上运行。循环步骤是，读代码，提出修复，应用补丁，运行测试。

Verification 是真正的 gate。只有当完整 CI 在沙箱中通过后，PR 才能被创建。系统会计算 coverage delta，如果下降超过阈值，PR 仍会打开，但会被打上 `needs-review` 标签。Agent 把 rationale 写进 PR description，同时创建一个 `@agent` 讨论线程，供 reviewer 继续追问。

安全依赖 GitHub 的两个不同能力面。App 提供短期 installation token，只授予 `workflows: read` 和窄化后的 repo contents / PR scopes。分支保护，不是 app 权限，负责强制 “no direct writes to `main`” 和 “no force-push”，而且 app 永远不应进入 bypass list。GitHub App 并没有针对 `.github/workflows` 的路径级只读权限，因此 agent 自身的文件编辑 allow-list 必须在 worker 侧强制执行。每仓库每日预算上限，比如每个 repo 每天最多 5 个 PR，每个 PR 最多 20 美元，则由 dispatcher 执行。

## Architecture

```
GitHub issue labeled `@agent fix` or PR comment
            |
            v
    GitHub App webhook -> AWS Lambda dispatcher
            |
            v
    ECS Fargate task (or GitHub Actions self-hosted runner)
       - pull repo
       - infer Dockerfile (language, package manager)
       - Daytona / E2B sandbox with target runtime
       - clone -> git worktree -> agent branch
            |
            v
    mini-swe-agent / SWE-agent v2 loop
       Claude Opus 4.7 or GPT-5.4-Codex
       tools: ripgrep, tree-sitter, read/edit, run_tests, git
            |
            v
    verify CI passes in-sandbox + coverage delta check
            |
            v (verified)
    git push + open PR via GitHub App
       PR body = rationale + diff summary + trace URL
       label: needs-review
            |
            v
    operator reviews; can @-mention agent for follow-ups
```

## Stack

- Trigger: GitHub App，细粒度 token，webhook receiver 使用 Lambda 或 Fly.io
- Worker: ECS Fargate task，或 GitHub Actions self-hosted runner
- Sandbox: Daytona devcontainer 或每任务一个 E2B sandbox
- Agent loop: 基于 Claude Opus 4.7 / GPT-5.4-Codex 的 mini-swe-agent 或 SWE-agent v2
- Retrieval: tree-sitter repo-map + ripgrep
- Verification: 沙箱内完整 CI + coverage delta gate
- Observability: Langfuse，每个 PR 的 trace archive 链接写进 PR body
- Budget: 每仓库每日美元上限，每仓库每日最大 PR 数量

## Build It

1. **GitHub App.** 使用细粒度 installation token，权限包括 issues read+write、pull_requests write、contents read+write、workflows read。分支保护，唯一能做到这一点的层面，负责强制 “no direct push to `main`” 和 “no force-push”，且 app 不进入 bypass list。由于 GitHub App 权限无法按路径切分，worker 必须通过 diff allow-list 强制 “不允许写 `.github/workflows`”。

2. **Webhook receiver.** Lambda function 接收 issue label / PR comment webhooks。只筛选带 `@agent fix this` 标签的事件。然后投递到 SQS。

3. **Dispatcher.** 从 SQS 弹出任务。对每个 repo 每天执行预算限制。用 repo URL、issue body 和一个全新的 Daytona sandbox 启动 ECS Fargate task。

4. **Environment inference.** 检测语言，Python、Node、Go、Rust，以及 package manager，uv、pnpm、go mod、cargo。如果 repo 里没有 Dockerfile，就在线生成一个。

5. **Agent loop.** 使用 mini-swe-agent 或 SWE-agent v2，模型选 Claude Opus 4.7。Tools 包括 ripgrep、tree-sitter repo-map、read_file、edit_file、run_tests、git。硬性上限是 20 美元、30 分钟 wall-clock 和 30 个 agent turns。

6. **Verification.** 循环结束后，在沙箱中运行完整测试套件。用 jacoco 或 coverage.py 计算 coverage delta。若 CI 为红，则停止，不创建 PR。若 coverage 降低超过 2%，则创建带 `needs-review` 标签的 PR。

7. **PR posting.** 推送 agent branch。通过 GitHub API 打开 PR，正文包含 title、rationale、diff summary、trace URL、cost 和 turns。

8. **Credential hygiene.** Worker 使用短期 GitHub App installation token。日志归档前要做 secret scrub。

9. **Eval.** 准备 30 个不同难度的内部种子 issues。测量 pass rate、PR quality，diff size、style、coverage，cost 与 latency。再把同一组问题与 Cursor Background Agents 和 AWS Remote SWE Agents 做对比。

## Use It

```
# on github.com
  - user labels issue #842 with `@agent fix this`
  - PR #1903 appears 14 minutes later
  - body:
    > Fixed NPE in widget.dedupe() caused by null comparator entry.
    > Added regression test widget_test.go::TestDedupeNullComparator.
    > Coverage delta: +0.12%
    > Turns: 7  Cost: $1.80  Trace: langfuse:...
    > Label: needs-review
```

## Ship It

`outputs/skill-issue-to-pr.md` 是可交付成果。它是一套 GitHub App + 异步云端 worker，可以把带标签的 issues 转成带预算边界和受限凭据的 review-ready PRs。

| Weight | Criterion | How it is measured |
|:-:|---|---|
| 25 | Pass rate on 30 issues | 端到端成功率，CI 变绿且 coverage 合格 |
| 20 | PR quality | Diff size、coverage delta、style conformance |
| 20 | Cost and latency per resolved issue | 每个 PR 的美元成本与墙钟时间 |
| 20 | Safety | Scoped token、每仓库预算、no force-push、credential hygiene |
| 15 | Operator UX | Rationale comments、retry affordance、`@` 提及后的追问体验 |
| **100** | | |

## Exercises

1. 增加 “fix flaky test” 模式。标签 `@agent stabilize-flake TestX` 会在沙箱里把该测试跑 50 次，并提出一个最小修复，使其稳定。

2. 在三个共享 issues 上与 Cursor Background Agents 做成本对比。报告哪些工具在哪些场景更强。

3. 实现一个 budget dashboard。按仓库统计每日成本，也按用户统计成本。对异常值发出告警。

4. 构建 “dry-run” 模式，不跑 CI，只开一个 draft PR，让 reviewer 先低成本审查计划。

5. 增加 retention policy。超过 7 天未合并的 PR branches 自动删除。

## Key Terms

| Term | What people say | What it actually means |
|------|-----------------|------------------------|
| GitHub App | “Scoped bot identity” | 带细粒度权限和短期 installation token 的应用身份 |
| Async cloud agent | “Background agent” | 在云端沙箱中运行的非交互式 worker，而不是终端中的 agent |
| Environment inference | “Dockerfile synthesis” | 检测语言和包管理器，并在缺失时生成 Dockerfile |
| Verification | “CI-in-sandbox” | 在 worker 内部跑完整测试套件，再决定是否开 PR |
| Coverage delta | “Coverage preservation” | 从 base 到 agent branch 的测试覆盖率变化 |
| Per-repo budget | “Daily ceiling” | 由 dispatcher 强制执行的每日美元上限与 PR 数上限 |
| Rationale | “PR body explanation” | Agent 对改动内容与原因的总结，必须写在 PR body 中 |

## Further Reading

- [AWS Remote SWE Agents](https://github.com/aws-samples/remote-swe-agents) — 经典的异步云端 agent 参考
- [SWE-agent](https://github.com/SWE-agent/SWE-agent) — CLI 参考实现
- [Cursor Background Agents](https://docs.cursor.com/background-agent) — 商业替代方案
- [OpenAI Codex (cloud)](https://openai.com/codex) — 托管竞品
- [Google Jules](https://jules.google) — Google 的托管版本
- [Factory Droids](https://www.factory.ai) — 另一种商业参考
- [GitHub App documentation](https://docs.github.com/en/apps) — 受限 bot 身份文档
- [Daytona cloud sandboxes](https://daytona.io) — 沙箱参考
