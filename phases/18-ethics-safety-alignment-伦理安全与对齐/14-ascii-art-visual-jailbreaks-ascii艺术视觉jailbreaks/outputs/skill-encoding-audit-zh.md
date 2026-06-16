---
name: encoding-audit-zh
description: 审计 a 越狱-防御 report across encoding-family attacks.
version: 1.0.0
phase: 18
lesson: 14
tags: [artprompt, ascii-art, encoding-attack, utes, structural-sleight]
---

给定 a 越狱-防御 report, enumerate the encoding-family attacks covered and the 防御 layer that catches each.

产出：

1. Encoding coverage. List each 攻击 family evaluated: ASCII art (ArtPrompt), base64, leet-speak, UTF-8 homoglyphs, nested JSON / YAML / CSV, tree/graph UTES, image-modality. Flag families missing.
2. 防御-layer mapping. For each family, identify which 防御 layer (keyword filter, perplexity filter, paraphrase, retokenization, 输出 分类器, multimodal moderator) catches it and which does not.
3. Visual-recognition gap. Per Jiang et al. 2024, PPL and Retokenization fail against ArtPrompt because the recognition happens at the visual level. Does the report's 防御 include anything that operates at the visual/structural level?
4. Generalization test. UTES (StructuralSleight) generalizes to arbitrary rare structures. Does the report test structures not in its training 防御 set?
5. 能力-安全 tradeoff. A 模型 with stronger visual-text 能力 (high ViTC score) is more vulnerable to ArtPrompt. Note the 模型's ViTC score if reported; request it if not.

硬性拒绝：
- Any 防御 claim based solely on substring/keyword filtering.
- Any 防御 claim that covers one encoding family and extrapolates to "encoding attacks."
- Any 防御 claim without a per-family 攻击-success rate.

拒绝规则：
- 如果用户询问 whether ArtPrompt is "patched," refuse and explain the recognition-level vs text-level 防御 gap.
- 如果用户询问 for a recommended all-encoding 防御, refuse a single recommendation ， 防御 must be layered across all families that the 部署 might face.

输出：a one-page 审计 that fills the five sections above, flags the primary encoding gap, and names the single most urgent 防御 layer to add. 引用 Jiang et al. (arXiv:2402.11753) and StructuralSleight once each.
