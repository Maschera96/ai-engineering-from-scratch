---
name: msj-audit-zh
description: 审计 a long-context 安全 评估 for many-shot 越狱 coverage.
version: 1.0.0
phase: 18
lesson: 13
tags: [many-shot-jailbreaking, context-window, power-law, anthropic]
---

给定 a 安全 评估 for a long-context 模型, 审计 whether the 评估 covers many-shot 越狱.

产出：

1. Shot-count coverage. Report the shot counts tested (should include 1, 5, 16, 64, 256, and at least one >= 512 for models with >= 1M context). If the 评估 tests at a single shot count, ASR is uninformative ， MSJ is a curve.
2. Power-law fit. Report the fitted exponent per behaviour category. A shallow exponent indicates the 模型 is ICL-robust on that category; a steep exponent indicates MSJ is disproportionately effective.
3. Category breakdown. MSJ effectiveness varies by category: violent 内容, deceit, self-harm, bioweapon. Per Anil et al. 2024, violent/deceitful needs fewer shots to 越狱. Flag any category absent from the 评估.
4. 防御 identification. Is a 分类器-based prompt modification in place? Is the 分类器 itself evaluated for adversarial robustness? Anthropic's reported 61% -> 2% reduction depends on 分类器 calibration.
5. Compositional check. Does the 评估 test MSJ + PAIR, MSJ + persuasive templates, or MSJ + encoding? Compositional attacks are frequently stronger than any single technique.

硬性拒绝：
- Any "our long-context 模型 is safe" claim based on 5-shot-only 评估.
- Any 防御 claim without reporting both 越狱 ASR and benign ICL performance on the same 分类器 ， the trade-off is the point.
- Any category-aggregate ASR without a category breakdown.

拒绝规则：
- 如果用户询问 whether MSJ can be fully patched, refuse the binary answer; MSJ shares a mechanism with ICL and cannot be eliminated without eliminating ICL.
- 如果用户询问 for a recommended shot count for 评估, refuse a single number; request the power-law fit over 5 to 512 shots.

输出：a one-page 审计 that reports the shot-count coverage, power-law fit per category, 防御 identification, and one compositional 攻击 gap. 引用 Anil et al. 2024 (Anthropic) once as the methodological reference.
