# Skills 与智能体 SDK —— Anthropic Skills、AGENTS.md、OpenAI Apps SDK

> MCP 说的是“存在哪些工具”。Skills 说的是“如何完成一项任务”。2026 年的技术栈把两者叠加在一起。Anthropic 的 Agent Skills（开放标准，2025 年 12 月）以 SKILL.md 形式发布，采用渐进式披露。OpenAI 的 Apps SDK 是 MCP 加上小部件元数据。AGENTS.md（如今已在 60,000+ 个仓库中使用）位于仓库根目录，作为项目级的智能体上下文。本课会明确各自覆盖的范围，并构建一个能在多个智能体间通用的最小化 SKILL.md + AGENTS.md 组合包。

**类型：** Learn
**语言：** Python（标准库，SKILL.md 解析器与加载器）
**前置条件：** 阶段 13 · 07（MCP 服务器）
**时长：** 约 45 分钟

## 学习目标

- 区分三个层级：AGENTS.md（项目上下文）、SKILL.md（可复用的专业知识）、MCP（工具）。
- 编写带有 YAML frontmatter 和渐进式披露的 SKILL.md。
- 以文件系统方式将技能加载进智能体运行时。
- 把一个技能与一个 MCP 服务器和一个 AGENTS.md 组合起来，使同一个包能在 Claude Code、Cursor 和 Codex 中工作。

## 问题

一位工程师把编写发布说明的工作流提炼成一个多步骤提示词：“读取最新合并的 PR。按领域分组。逐个总结。按照团队风格写一条变更日志条目。发布到 Slack 草稿。”他们把它放进了团队的 Notion 文档。

现在他们想在 Claude Code、Cursor 和 Codex CLI 中使用这个工作流。每个智能体加载指令的方式各不相同：Claude Code 用斜杠命令，Cursor 用 rules，Codex 用 `.codex.md`。于是这位工程师把工作流复制了三份，并维护这三份副本。

AGENTS.md 和 SKILL.md 结合起来解决了这个问题：

- **AGENTS.md** 位于仓库根目录。每个兼容的智能体在会话开始时都会读取它。“这个项目如何运作？有哪些约定？哪些命令用来跑测试？”
- **SKILL.md** 是一个可移植的组合包：YAML frontmatter（name、description）+ markdown 正文 + 可选资源。支持技能的智能体会按需通过名称加载它们。
- **MCP**（阶段 13 · 06-14）负责处理技能需要调用的工具。

三个层级，一个可移植的产物。

## 概念

### AGENTS.md (agents.md)

于 2025 年末推出，到 2026 年 4 月已被 60,000+ 个仓库采用。仓库根目录下的一个文件。格式：

```markdown
# Project: my-service

## Conventions
- TypeScript with strict mode.
- Use Pydantic for models on the Python side.
- Tests run with `pnpm test`.

## Build and run
- `pnpm dev` for local dev server.
- `pnpm build` for production bundle.
```

智能体在会话开始时读取它，并据此校准自己在该项目中的行为。2026 年每一个编码智能体都支持 AGENTS.md：Claude Code、Cursor、Codex、Copilot Workspace、opencode、Windsurf、Zed。

### SKILL.md 格式

Anthropic 的 Agent Skills（于 2025 年 12 月作为开放标准发布）：

```markdown
---
name: release-notes-writer
description: Write a changelog entry for the latest merged PRs following this project's style.
---

# Release notes writer

When invoked, run these steps:

1. List PRs merged since the last tag. Use `gh pr list --base main --state merged`.
2. Group by label: feature, fix, chore, docs.
3. For each PR in each group, write one line: `- <title> (#<num>)`.
4. Draft the release notes and stage them in CHANGELOG.md.

If the user says "ship", run `git tag vX.Y.Z` and `gh release create`.

## Notes

- Never include commits without a PR.
- Skip "chore" entries from the public changelog.
```

Frontmatter 声明技能的身份。正文是技能加载时展示给模型的提示词。

### 渐进式披露

技能可以引用子资源，智能体仅在需要时才去获取它们。示例：

```
skills/
  release-notes-writer/
    SKILL.md
    style-guide.md
    template.md
    scripts/
      generate.sh
```

SKILL.md 写明“风格规则见 style-guide.md”。智能体仅在技能实际运行时才拉取 style-guide.md。这样可以避免用模型可能用不到的细节来撑大提示词。

### 文件系统发现

智能体运行时会扫描已知目录以查找 SKILL.md 文件：

- `~/.anthropic/skills/*/SKILL.md`
- 项目 `./skills/*/SKILL.md`
- `~/.claude/skills/*/SKILL.md`

加载依据是文件夹名和 frontmatter 中的 `name`。Claude Code、Anthropic Claude Agent SDK 以及 SkillKit（跨智能体）都遵循这一模式。

### Anthropic Claude Agent SDK

`@anthropic-ai/claude-agent-sdk`（TypeScript）和 `claude-agent-sdk`（Python）在会话开始时加载技能，并把它们作为可调用的“智能体”暴露在运行时内部。当用户调用某个技能时，智能体循环会派发到该技能。

### OpenAI Apps SDK

于 2025 年 10 月推出；直接构建在 MCP 之上。它把 OpenAI 此前的 Connectors 和 Custom GPT Actions 统一到单一的开发者界面之下。一个 Apps SDK 应用是：

- 一个 MCP 服务器（工具、资源、提示词）。
- 外加为 ChatGPT 界面提供的小部件元数据。
- 外加一个可选的、用于交互界面的 MCP Apps `ui://` 资源。

相同的协议，更丰富的用户体验。

### 通过 SkillKit 实现跨智能体可移植

像 SkillKit 这样的工具及类似的跨智能体分发层，会把单个 SKILL.md 翻译成 32+ 个 AI 智能体（Claude Code、Cursor、Codex、Gemini CLI、OpenCode 等）各自的原生格式。一处真相来源，多方消费。

### 三层技术栈

| 层级 | 文件 | 加载时机 | 用途 |
|-------|------|-------------|---------|
| AGENTS.md | 仓库根目录 | 会话开始 | 项目级约定 |
| SKILL.md | skills 目录 | 调用技能时 | 可复用的工作流 |
| MCP 服务器 | 外部进程 | 需要工具时 | 可调用的操作 |

三者协同组合：智能体在会话开始时读取 AGENTS.md，用户调用某个技能，技能的指令中包含 MCP 工具调用，智能体通过 MCP 客户端进行派发。

## 上手实践

`code/main.py` 提供了一个基于标准库的 SKILL.md 解析器与加载器。它在 `./skills/` 下发现技能，解析 YAML frontmatter 加 markdown 正文，并生成一个以技能名为键的字典。随后它模拟一个智能体循环，按名称调用 `release-notes-writer`。

重点关注：

- 用最小化的标准库解析器解析 YAML frontmatter（不依赖 `pyyaml`）。
- 技能正文原样存储；智能体在调用时将其前置到系统提示词中。
- 通过一个 `read_subresource` 函数演示渐进式披露，该函数按需拉取被引用的文件。

## 交付实践

本课会产出 `outputs/skill-agent-bundle.md`。给定一个工作流，该技能会生成组合后的 SKILL.md + AGENTS.md + MCP 服务器蓝图组合包，可在多个智能体间移植。

## 练习

1. 运行 `code/main.py`。在 `skills/` 下新增第二个技能，并确认加载器能识别它。

2. 为本课程仓库编写一份 AGENTS.md。包含测试命令、风格约定，以及阶段 13 的心智模型。

3. 把你团队内部文档中的一个多步骤工作流移植成一个 SKILL.md。验证它能在 Claude Code 中加载。

4. 手动把该技能翻译成 Cursor 和 Codex 各自的原生 rule 格式。统计这些格式之间的差异——这正是 SkillKit 所自动化的翻译面。

5. 阅读 Anthropic 的 Agent Skills 博客文章。找出 Claude Agent SDK 中本课加载器未覆盖的一个特性。（提示：智能体子调用。）

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| SKILL.md | “技能文件” | YAML frontmatter 加 markdown 正文，由智能体运行时加载 |
| AGENTS.md | “仓库根目录的智能体上下文” | 在会话开始时读取的项目级约定文件 |
| 渐进式披露 | “惰性加载子资源” | 技能正文引用的文件仅在需要时才被拉取 |
| Frontmatter | “顶部的 YAML 块” | 位于 `---` 分隔符内的元数据（name、description） |
| Claude Agent SDK | “Anthropic 的技能运行时” | `@anthropic-ai/claude-agent-sdk`，加载技能并路由 |
| OpenAI Apps SDK | “MCP + 小部件元数据” | OpenAI 构建在 MCP 加 ChatGPT 界面钩子之上的开发者界面 |
| 技能发现 | “文件系统扫描” | 遍历已知目录查找 SKILL.md，以名称为键 |
| 跨智能体可移植性 | “一个技能多个智能体” | 通过类 SkillKit 工具将一个 SKILL.md 翻译给 32+ 个智能体 |
| Agent Skill | “可移植的专业知识” | MCP 工具概念之外的可复用任务模板 |
| Apps SDK | “MCP 加 ChatGPT 界面” | 在 MCP 之上统一的 Connectors 与 Custom GPTs |

## 延伸阅读

- [Anthropic — Agent Skills announcement](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) — 2025 年 12 月发布
- [Anthropic — Agent Skills docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) — SKILL.md 格式参考
- [OpenAI — Apps SDK](https://developers.openai.com/apps-sdk) — 面向 ChatGPT 的基于 MCP 的开发者平台
- [agents.md](https://agents.md/) — AGENTS.md 格式与采用清单
- [Anthropic — anthropics/skills GitHub](https://github.com/anthropics/skills) — 官方技能示例
