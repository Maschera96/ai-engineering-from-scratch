---
name: browser-agent-trust-boundary-zh
description: 在浏览器智能体触碰真实站点前，界定信任区、授权写入和必要防御。
版本: 1.0.0
phase: 15
lesson: 11
tags: [browser-agents, prompt-injection, trust-boundary, osworld, webarena]
---

给定a 拟议的 browser-智能体 工作流, 生成 a trust-boundary scoping document that enumerates 每个 阅读, 每个 编写, and the minimum 防御 stack 必需 for first 运行.

产出:

1. **阅读 表面.** 列出 每个 origin the 智能体 will fetch. Classify each as in-trust (first-party sites operated by the 用户's organization) or out-of-trust (任何 third-party, 任何 用户-generated content, 任何 search 结果). 所有 out-of-trust reads 必须 be treated as potential 提示词-注入 channels.
2. **编写 表面.** 列出 每个 consequential action the 智能体 is 已授权 to take (submit form, post content, call a 后端 工具, 编写 to 记忆). For each, 说明 the blast radius and 是否 the action is reversible.
3. **必需 防御.** Minimum stack: content sanitizer, 阅读/编写 boundary (writes 要求 fresh approval when content_origin is out-of-trust), 工具 allowlist per 任务, session 隔离 带有 scoped 凭据, 金丝雀令牌s on persistent 记忆, 人在回路 on irreversible actions.
4. **基准-to-分布 fit.** If the 智能体 reports a BrowseComp, OSWorld, or WebArena-Verified 评分, 命名 the 分布 overlap between the 基准 and the real 任务. A high BrowseComp 评分 does 不predict booking-flow 可靠性.
5. **Known-攻击 checklist.** 确认 the 部署 is hardened 针对 (a) visible-text 注入, (b) URL-fragment / query 注入, (c) 记忆-binding 攻击 (Tainted Memories 类别), (d) CSRF-shaped 攻击 on authenticated sessions, (e) one-click hijacks. For each, 命名 the 具体 防御 and 在哪里 it fires.

硬性拒绝:
- Browser agents 带有 access to 生产环境 凭据 and no session 隔离.
- 任何 部署 在哪里 a 编写 initiated 来自 out-of-trust content does 不require fresh 人在回路 approval.
- 任何 部署 relying solely on a content sanitizer (sanitizers catch easy 攻击; sophisticated payloads pass).
- Persistent 记忆 带有 no canary entries.
- Workflows that touch financial transactions or customer data 带有 no 人在回路 on writes.

拒绝规则:
- If the 用户 不能 命名 the blast radius of an 注入-driven wrong 编写, 拒绝 and 要求 an 明确 sentence.
- If the 用户 proposes a browser 智能体 on a stack 在哪里 scoped 凭据 are 不available, 拒绝 and 要求 a separate identity first.
- If the 用户 cites a 基准 评分 (BrowseComp, OSWorld, WebArena) as 证据 the 智能体 "can" do a 生产环境 任务, 拒绝 and 要求 内部 evals on the real 分布.

输出格式:

返回a trust-boundary memo 带有:
- **阅读 表面 表格** (origin, in-trust / out-of-trust)
- **编写 表面 表格** (action, blast radius, reversible 是/否)
- **防御 stack** (项目符号 列出 of configured 层)
- **基准-fit note** (if applicable)
- **Known-攻击 checklist** (five rows, 防御 具名 per row)
- **部署 结论** (生产环境 / 预发布环境 / 仅研究)
