---
name: classifier-stack-audit-zh
description: 审计输入输出分类器栈，包含模型、分类法、输入护栏、输出护栏、对话护栏和对抗攻击缺口。
版本: 1.0.0
phase: 15
lesson: 18
tags: [llama-guard, nemo-guardrails, input-rails, output-rails, colang, adversarial-attacks]
---

给定a 部署's 分类器 stack (Llama Guard 版本, NeMo Guardrails config, custom 分类器s, 规范化 steps), 审计 it 针对 the 2026 reference and 标出 攻击 表面 the stack does 不cover.

产出:

1. **Model inventory.** 列出 the 分类器s in use. Llama Guard 3 (8B / 1B-INT4) vs Llama Guard 4 (多模态, S1–S14). NeMo Guardrails 版本. 任何 custom 分类器s. If the 部署 accepts images, 确认 the 分类器 is 多模态.
2. **分类法 mapping.** 映射 declared business 类别 onto the 分类器's 分类法. 每个 类别 the 操作员 cares about 必须 映射 to a 分类器 类别; unmapped 类别 are unguarded.
3. **Rail coverage.** 确认 输入护栏 fire 之前 the 模型 turn and 输出护栏 fire 之前 the response ships. 对话 护栏 (Colang in NeMo) enforce cross-turn constraints. 单一-turn 分类器s 不能 catch multi-turn 攻击.
4. **规范化.** 确认 输入 are NFKC-normalized, homoglyph-mapped, and have zero-width / variation-selector characters stripped 之前 分类. Raw-byte 分类 is a 100% ASR target for Emoji Smuggling (Huang et al. 2025).
5. **攻击-corpus coverage.** For each documented 攻击 (emoji smuggling, homoglyph, in-上下文 redirection, semantic paraphrase), 命名 the 具体 防御 in the stack. 分类器-只 防御 fails this 审计; layering 带有 宪法 (Lesson 17) and runtime (Lessons 10, 13, 14) is 必需.

硬性拒绝:
- Deployments using a text-只 分类器 on 多模态 输入.
- Deployments 带有 no 规范化 step.
- Deployments 带有 输入护栏 只 (no 输出护栏 on sensitive-类别 输出).
- Stack treating the 分类器 as the 单一 安全 层.
- ASR 声明 the 操作员 不能 reproduce on their own 分布.

拒绝规则:
- If the 用户's declared 类别 do 不map 进入 the 分类器's 分类法, 拒绝 and 要求 a mapping first. Unmapped = unguarded.
- If the 部署 cites Llama Guard 3 ASR numbers on a 多模态 输入 表面, 拒绝 and 要求 Llama Guard 4 or a 多模态 分类器.
- If the 用户 treats the 分类器 层 as sufficient in a high-风险 设置, 拒绝. EU AI Act Article 14 (Lesson 15) expects 人类 oversight on top.

输出格式:

返回a 分类器 审计 带有:
- **Model inventory** (命名, 版本, modality)
- **分类法 mapping** (操作员 类别 → 分类器 类别)
- **Rail coverage** (输入 / 输出 / 对话; firing 之前/之后 模型)
- **规范化 note** (NFKC 是/否, homoglyph 是/否, zero-width strip 是/否)
- **攻击-corpus coverage** (攻击 → 防御)
- **层完整性** (分类器 + 宪法 + runtime; three 必需)
- **就绪度** (生产环境 / 预发布环境 / 仅研究)
