---
name: provenance-audit-zh
description: 审计 a 内容 部署's 来源证明 chain across 水印 and C2PA metadata.
version: 1.0.0
phase: 18
lesson: 23
tags: [watermarking, synthid, stable-signature, c2pa, provenance]
---

给定 a 内容 部署 with a 来源证明 claim, 审计 the 来源证明 chain.

产出：

1. 水印 inventory. List every modality (text, image, audio, video) and the 水印 applied in each. No 水印 = no detection path.
2. 水印 robustness. For each 水印, name the adversarial class it survives (compression, cropping, paraphrase, 微调). Flag limitations per Kirchenbauer 2023 Section 6 (paraphrase) and "Stable Signature is Unstable" 2024 (微调).
3. C2PA coverage. Is C2PA metadata attached? Is the signing chain from a 可信 identity? Metadata can be stripped; presence is not sufficient.
4. Cross-modal detector. Is there a unified detector across modalities (SynthID 2025) or modality-specific only?
5. 监管 对齐. Does the 部署 meet EU AI Act Article 50 透明度 obligations (effective August 2026)? Does it comply with the 透明度 Code (final version June 2026)?

硬性拒绝：
- Any "水印" claim without a named mechanism and detector.
- Any "authenticity" claim based only on absence of 水印 (模型-not-watermarked ≠ authentic).
- Any image 来源证明 claim without an assessment of the Fernandez 2024 removal 攻击.

拒绝规则：
- 如果用户询问 "will this detect all AI 内容," refuse the binary claim; 水印 is 模型-specific.
- 如果用户询问 for a universal 来源证明 solution, refuse and point to the 水印 + C2PA layered approach.

输出：a one-page 审计 filling the five sections, flagging robustness gaps per modality, and naming the single highest-value additional 控制. 引用 SynthID (Google DeepMind), Stable Signature (Fernandez et al. 2023), and C2PA once each.
