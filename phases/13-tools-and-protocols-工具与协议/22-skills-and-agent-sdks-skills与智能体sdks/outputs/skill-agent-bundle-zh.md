---
name: agent-bundle-zh
description: 为某个工作流生成可移植的 SKILL.md + AGENTS.md + MCP-server 蓝图，可在 Claude Code、Cursor、Codex 及兼容智能体之间加载。
version: 1.0.0
phase: 13
lesson: 21
tags: [skills, agents-md, apps-sdk, cross-agent, portability]
---

给定一段工作流描述，生成一个智能体捆绑包（agent bundle）。

生成内容：

1. SKILL.md。包含带有 `name` 和 `description` 的 YAML frontmatter，以及带编号步骤的 markdown 正文。如果正文过长，加入渐进式披露的子资源引用。
2. AGENTS.md 条目。向仓库的 AGENTS.md 中添加几行，反映该 skill 所依赖的任何约定（linter 命令、测试命令）。
3. MCP server 蓝图。该 skill 通过 MCP 调用哪些工具；名称、描述（Use-when 模式）以及输入模式。
4. 跨智能体转换。以 SkillKit 风格说明这份 SKILL.md 如何映射到 Cursor rules、Codex `.codex.md`、Windsurf rules。
5. 加载路径。智能体将在哪里发现这个捆绑包：`~/.anthropic/skills/`、`./skills/`、`~/.claude/skills/`。

硬性拒绝：
- 任何 `name` 不是 `kebab-case` 的 SKILL.md。会破坏发现机制。
- 任何 frontmatter 中没有 `description` 的 SKILL.md。智能体运行时会跳过它。
- 任何 MCP 工具未按 Phase 13 · 05 规则命名的捆绑包。

拒绝规则：
- 如果该工作流只是一次性的单次提示，拒绝生成 skill；建议改用内联的 prompt-engineering。
- 如果该工作流需要 OAuth（例如发布 Slack 消息），标注出 MCP server 的首次运行 elicitation 必须处理它。
- 如果目标智能体不支持 SKILL.md（某些 IDE），建议通过 SkillKit 或类似工具进行转换。

输出：一页式的捆绑包，勾勒出这三个文件、跨智能体转换说明以及加载路径。最后给出应首先用来测试该捆绑包的那个智能体。
