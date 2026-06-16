---
name: constitution-review-zh
description: 审计部署的宪法层，包括硬编码禁令、软编码默认值、操作员边界和四级层级解析。
版本: 1.0.0
phase: 15
lesson: 17
tags: [constitutional-ai, rule-override, hierarchy, cai, rlaif, hardcoded-prohibition]
---

给定a 部署's 宪法式 层 (系统 提示词, 操作员 config, declared 原则), 审计 it 针对 the Claude 宪法 reference and 标出 缺失 硬编码 禁令, 含糊 原则, or misordered tiers.

产出:

1. **硬编码 禁令 inventory.** 列出 每个 禁令 that 必须 不bend regardless of 操作员 or 用户 instruction. Minimum floor: bioweapons / CBRN uplift, CSAM, critical infrastructure 攻击 规划, false-identity-when-asked. Additions are 部署-具体 (e.g., financial services adds 具体 fraud 禁令).
2. **软编码默认值.** 列出 每个 behaviour the 操作员 can adjust. For each, 说明 the declared bound. An "adjustable" 设置 带有 no bound is a back-door override.
3. **层级顺序ing.** 确认 the resolution order is: 安全 > 伦理 > 准则 > 有用性. If 有用性 ever wins over 伦理 in the implemented resolver, 标出 as a 部署 break.
4. **原则歧义标记.** 识别 任何 原则 whose text leaves room for materially different interpretations. Ambiguity compounds over training cycles (原则 drift).
5. **层完整性.** 确认 运行时层控制 (Lessons 10, 13, 14) are 存在 in addition to the 宪法式 层. 宪法 alone is insufficient; runtime alone is insufficient.

硬性拒绝:
- 部署缺少任何硬编码禁令层。
- 操作员配置声明可覆盖硬编码禁令，即使只是改名也不允许。
- 把有用性放在伦理之上的层级顺序。
- 原则文本过于宽泛，无法评估 ("be good").
- Treating 宪法式 AI as a replacement for 运行时控制.

拒绝规则:
- If the 用户 names a 硬编码 禁令 but 不能 point to a runtime-层 backstop for it, 标出 the 部署 as 单一-层 and 拒绝 生产环境.
- If the 操作员 config includes an adjustable "安全" 设置 带有 no declared bound, 拒绝.
- If the 用户 treats the 2023 participatory-宪法 findings as actionable in the current 部署, check: the 2026 宪法 did 不incorporate them, so "inherits democratically" is a 声明 the 部署 不能 back up.

输出格式:

返回a 宪法式 审计 带有:
- **硬编码底线** (禁令, enforcement 层: 权重 / 推理 / 两者)
- **软编码默认值** (设置, 操作员 bound, 用户-visible 是/否)
- **层级顺序** (已列出; 已确认 安全 > 伦理 > 准则 > 有用性)
- **歧义标记** (原则, 具体 ambiguity, 拟议的 收紧)
- **层完整性** (宪法式 是/否, 运行时控制 是/否, 两者 必需)
- **就绪度** (生产环境 / 预发布环境 / 仅研究)
