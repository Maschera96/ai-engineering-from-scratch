# 浏览器智能体与长周期 Web 任务

> ChatGPT 智能体 (July 2025) merged 操作员 and deep 研究 进入 one browser/terminal 智能体 and 设置 BrowseComp SOTA at 68.9%. OpenAI shut 操作员 down August 31, 2025 — consolidation at the product 层. Anthropic's Vercept acquisition moved Claude Sonnet on OSWorld 来自 under 15% to 72.5%. WebArena-Verified (ServiceNow, ICLR 2026) fixed 11.3 percentage points of false-negative rate in the original WebArena and shipped the 258-任务 Hard subset. The numbers are real. So is the 攻击 表面: OpenAI's head of preparedness stated publicly that indirect 提示词 注入 进入 browser agents "is 不a bug that can be fully patched." Documented 2025–2026 攻击: Tainted Memories (Atlas CSRF), HashJack (Cato Networks), and one-click hijacks in Perplexity Comet.

**Type:** Learn
**Languages:** Python (stdlib, indirect 提示词-注入 攻击 surface model)
**Prerequisites:** Phase 15 · 10 (Permission modes), Phase 15 · 01 (Long-horizon agents)
**Time:** ~45 minutes

## 问题

一个browser 智能体 is a 长周期智能体 that reads 不可信 content and takes consequential actions. 每个 page the 智能体 visits is an 输入 the 用户 did 不write. 每个 form on 每个 page is a potential command 渠道. The 2025–2026 攻击 corpus shows this is 不hypothetical: Tainted Memories lets an attacker bind malicious instructions to the 智能体's 记忆 via a crafted page; HashJack hides commands in URL fragments the 智能体 visits; Perplexity Comet hijacks hit in a 单一 click.

这个defensive picture is uncomfortable. OpenAI's head of preparedness said the quiet part loud: indirect 提示词 注入 "is 不a bug that can be fully patched." This is because the 攻击 lives in the 智能体's reading-vs-acting boundary, 哪个 is architecturally fuzzy — 每个 词元 the 模型 reads could, in 原则, be 阅读 as an instruction.

这lesson names the 攻击 表面, names the 基准 landscape (BrowseComp, OSWorld, WebArena-Verified), and models a minimal indirect-提示词-注入 scenario so you can reason about real 防御 in Lessons 14 and 18.

## 概念

### 这个2026 landscape, in 一段话 per 系统

**ChatGPT 智能体 (OpenAI).** Launched July 2025. Unifies 操作员 (browsing) and Deep Research (multi-hour 研究). Shut down the standalone 操作员 August 31, 2025. SOTA on BrowseComp at 68.9%; strong numbers on OSWorld and WebArena-Verified.

**Claude Sonnet + Vercept (Anthropic).** Anthropic's Vercept acquisition focused on computer-use 能力. Moved Claude Sonnet on OSWorld 来自 <15% to 72.5%. Claude Computer Use ships as a 工具 API.

**Gemini 3 Pro 带有 Browser Use (DeepMind).** Browser Use integration ships computer-use 控制; FSF v3 (April 2026, Lesson 20) tracks autonomy in the ML R&D domain specifically.

**WebArena-Verified (ServiceNow, ICLR 2026).** Fixes a well-documented problem: the original WebArena had ~11.3% false-negative rate (任务 marked failed that were actually solved). The Verified release re-grades 带有 人类-curated success criteria and adds a 258-任务 Hard subset (ICLR 2026 论文, openreview.net/论坛?id=94tlGxmqkN).

### BrowseComp vs OSWorld vs WebArena

| 基准 | 内容 it measures | Horizon |
|---|---|---|
| BrowseComp | Finding 具体 facts on the open web under time pressure | minutes |
| OSWorld | Agent operating a full desktop (mouse, keyboard, shell) | tens of minutes |
| WebArena-Verified | Transactional web 任务 in simulated sites | minutes |
| Hard subset | WebArena-Verified 任务 带有 multi-page 说明 transitions | tens of minutes |

Different axes. A high BrowseComp 评分 says the 智能体 finds facts; it does 不say the 智能体 can book a flight. The OSWorld 评分 is closer to "does it work on my desktop." WebArena-Verified is closer to "can it finish a flow." 任何 生产环境 decision needs the 基准 that matches the 任务 分布.

### 这个attack 表面, 具名

1. **Indirect 提示词 注入.** Untrusted page content contains instructions. The 智能体 reads them. The 智能体 executes them. Public examples: 2024 Kai Greshake et al., 2025 Tainted Memories 论文, 2026 HashJack (Cato Networks).
2. **URL fragment / query 注入.** The `#fragment` or query string of a crawled URL contains commands. Never rendered visibly; still 内部 the 智能体's 上下文.
3. **记忆-binding 攻击.** Page instructs the 智能体 to 编写 a persistent 记忆 (Lesson 12 covers 持久 说明). Next session, the 记忆 fires the payload 带有 no visible 触发器.
4. **CSRF-shaped 攻击 on authenticated sessions.** Tainted Memories 类别: 智能体 is logged in somewhere; attacker's page issues 说明-changing requests the 智能体 executes 带有 the 用户's cookies.
5. **One-click hijack.** A visually innocuous button rides a payload the 智能体 follows. Comet 类别.
6. **Content-Security-政策 holes in the 智能体's host 表面.** The rendering and 工具 层 can themselves be 攻击 vectors; the browser-in-a-browser-智能体 stack is wide.

### 原因 "不fully patchable"

这个attack is isomorphic to the 智能体's 能力. The 智能体 必须 阅读 不可信 content to do its job. 任何 content the 智能体 reads could contain instructions. 任何 instructions the 智能体 follows could be misaligned 带有 the 用户's actual request. 防御 (trust boundaries, 分类器s, 工具 allowlists, 人在回路 on consequential actions) raise the 成本 of the 攻击 and reduce its blast radius. They do 不close the 类别.

这is the same reasoning pattern as Lob's theorem (Lesson 8): the 智能体 不能 prove the next 词元 is safe; it can 只 设置 up a 系统 在哪里 unsafe 词元 are more detectable.

### 防御 posture that actually ships

- **阅读 / 编写 boundary.** Reading is never consequential. Writing (submitting a form, posting content, calling a 工具 带有 side effects) 要求 fresh 人类 approval if the initiating content came 来自 外部 the trust boundary.
- **工具 allowlist per 任务.** The 智能体 can browse; it 不能 initiate a wire transfer unless that 工具 was explicitly enabled for the 任务. Lesson 13 covers 预算.
- **Session 隔离.** Browser 智能体 sessions 运行 带有 scoped 凭据 只. No 生产环境 auth, no personal email. 日志 of 每个 HTTP request retained for 审计.
- **Content sanitizer.** Fetched HTML is stripped of known-bad patterns 之前 being concatenated 进入 the 模型 上下文. (Reduces the easy 攻击; does 不stop sophisticated payloads.)
- **人在回路 on consequential actions.** Propose-then-commit pattern (Lesson 15).
- **Canary 词元 on 记忆.** If a 记忆 entry fires, the 用户 sees it (Lesson 14).

## 使用它

`code/main.py` models a tiny browser-智能体 运行 针对 three synthetic pages. One page is benign, one has a direct 提示词-注入 blob in visible text, one has a URL-fragment 注入 (不visible but 内部 the 智能体's 上下文). The script shows (a) 什么 a naïve 智能体 would do, (b) 什么 a 阅读/编写 boundary catches, (c) 什么 a sanitizer catches, (d) 什么 neither catches.

## 交付它

`outputs/skill-browser-agent-trust-boundary.md` scopes a 拟议的 browser-智能体 部署: 哪个 trust zones it touches, 什么 it is 已授权 to 编写, and 哪个 防御 必须 be in place 之前 the first 运行.

## 练习

1. 运行 `code/main.py`. 识别 哪个 攻击 the sanitizer catches but the 阅读/编写 boundary does not, and 哪个 攻击 只 the 阅读/编写 boundary catches.

2. Extend the sanitizer to detect one 类别 of HashJack-style URL-fragment 注入. Measure the false-positive rate on benign URLs 带有 legitimate fragments.

3. Pick one real browser-智能体 工作流 you know (e.g., "book a flight"). 列出 每个 阅读 and 每个 编写. Mark 哪个 writes need 人在回路 and 为什么.

4. 阅读 the WebArena-Verified ICLR 2026 论文. 识别 one 类别 of 任务 在哪里 the original WebArena's scoring was unreliable and 解释 如何 the Verified subset resolves it.

5. 设计 a 记忆 canary for a browser-智能体 设置. 内容 would you store, 在哪里, and 什么 triggers the alarm?

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| Indirect 提示词 注入 | "Bad page text" | Untrusted content in a page the 智能体 reads contains instructions the 智能体 executes |
| Tainted Memories | "记忆 攻击" | Agent writes an attacker-supplied instruction to 持久 记忆; triggered next session |
| HashJack | "URL fragment 攻击" | Payload hidden in URL fragment / query string is in the 智能体's 上下文 but 不visibly rendered |
| One-click hijack | "Bad button" | Visible affordance rides a follow-on payload the 智能体 executes |
| BrowseComp | "Web search 基准" | Finding 具体 facts on the open web; minute-scale horizon |
| OSWorld | "Desktop 基准" | Full OS 控制; multi-step GUI 任务 |
| WebArena-Verified | "Fixed web-任务 基准" | ServiceNow's regraded WebArena 带有 Hard subset |
| 阅读/编写 boundary | "Side-effect gate" | Reading never consequential; writing 要求 fresh approval if content is out-of-trust |

## 延伸阅读

- [OpenAI — Introducing ChatGPT agent](https://openai.com/index/introducing-chatgpt-agent/) — merge of 操作员 and deep 研究; BrowseComp SOTA.
- [OpenAI — Computer-Using Agent](https://openai.com/index/computer-using-agent/) — the 操作员 lineage and the architecture that became ChatGPT 智能体.
- [Zhou et al. — WebArena](https://webarena.dev/) — the original 基准.
- [WebArena-Verified (OpenReview)](https://openreview.net/forum?id=94tlGxmqkN) — ICLR 2026 fixed-subset 论文.
- [Anthropic — Measuring agent autonomy in practice](https://www.anthropic.com/research/measuring-agent-autonomy) — includes 攻击-表面 discussion for computer-use agents.
