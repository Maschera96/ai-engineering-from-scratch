# 水印 ， SynthID, Stable Signature, C2PA

> Three technologies structure 2026 AI-generated-内容 来源证明. SynthID (Google DeepMind) ， image 水印 launched August 2023, text+video May 2024 (Gemini + Veo), text open-sourced October 2024 via Responsible GenAI Toolkit, unified multi-media detector November 2025 alongside Gemini 3 Pro. Text 水印 adjusts next-token sampling probabilities imperceptibly; image/video watermarks survive compression, cropping, filters, frame-rate changes. Stable Signature (Fernandez et al., ICCV 2023, arXiv:2303.15435) ， fine-tunes the latent diffusion decoder so every 输出 contains a fixed message; cropped (10% of 内容) generated images detected >90% at FPR<1e-6. Follow-up "Stable Signature is Unstable" (arXiv:2405.07145, May 2024) ， 微调 removes the 水印 while preserving quality. C2PA ， cryptographically signed, tamper-evident metadata standard (C2PA 2.2 Explainer 2025). 水印 and C2PA are complementary: metadata can be stripped but carries richer 来源证明; watermarks persist through transcoding but carry less information.

**类型:** 构建
**语言:** Python (stdlib, token-水印 embed + detect)
**先修要求:** Phase 10 · 04 (sampling), Phase 01 · 09 (information theory)
**时间:** ~75 分钟

## 学习目标

- 描述 token-level 水印 (SynthID-text style) and the mechanism by which it is detectable.
- 描述 Stable Signature and the 2024 removal 攻击 that broke it.
- 说明 C2PA's role and why it is complementary to 水印.
- 描述 the key limitations: 模型-specific signal, robustness under paraphrase, and meaning-preserving attacks (arXiv:2508.20228).

## 问题

2023-2024 saw deepfakes and AI-generated 内容 enter political and consumer contexts at scale. 水印 is the proposed technical 来源证明 signal: mark generations at creation time, detect them later. 2025 evidence: no 水印 is unconditionally robust, but layered with C2PA metadata the combination provides a usable 来源证明 story.

## 概念

### Text 水印 (SynthID-text style)

The Kirchenbauer et al. 2023 mechanism, productionized by Google:

1. At each decoding step, hash the previous K tokens to produce a pseudorandom partition of the vocabulary into "green" and "red" sets.
2. 偏差 sampling toward the green set by adding δ to green logits.
3. The generation contains more green tokens than chance would produce.

Detection: rehash each prefix, count green tokens in the generation, compute a z-score. The z-score is >0 for watermarked text, ~0 for human text.

Properties:
- Imperceptible to readers (δ is small enough that quality loss is minor).
- Detectable with access to the vocabulary partition function.
- Not robust to paraphrase ， rewriting the text destroys the signal.

SynthID-text is open-sourced October 2024 via Google's Responsible GenAI Toolkit.

### Stable Signature (image)

Fernandez et al. ICCV 2023. 微调 the latent diffusion decoder so every generated image contains a fixed binary message embedded in the latent representation. Detection is decoded from the latent with a neural decoder. Cropped (to 10% of 内容) images detected >90% at FPR<1e-6.

May 2024 "Stable Signature is Unstable" (arXiv:2405.07145): 微调 the decoder removes the 水印 while preserving image quality. Adversarial post-generation 微调 is cheap; the 水印's adversarial robustness is limited.

### SynthID unified detector (November 2025)

Alongside Gemini 3 Pro: a multi-media detector that reads SynthID signals from text, image, audio, and video in one API. Unifies the Google 来源证明 stack.

### C2PA

Coalition for 内容 来源证明 and Authenticity. Cryptographically signed tamper-evident metadata standard. C2PA 2.2 Explainer (2025). A C2PA manifest records 来源证明 claims (who created, when, what transformations) signed by the creator's key.

Complementary to 水印:
- Metadata can be stripped; watermarks cannot (easily).
- Metadata is rich (full 来源证明 chain); watermarks carry bits.
- C2PA depends on platform adoption; watermarks embed automatically.

Google integrates both in Search, Ads, and "About this image."

### Limitations

- **模型-specific.** SynthID watermarks generations from SynthID-enabled models. A generation from a 模型 without SynthID is not watermarked, so "no SynthID signal" is not proof of authenticity.
- **Paraphrase.** Text watermarks do not survive meaning-preserving paraphrase.
- **Transformation attacks.** arXiv:2508.20228 (2025) shows meaning-preserving attacks that destroy both text watermarks and many image watermarks.
- **微调 removal.** Per "Stable Signature is Unstable," post-generation 微调 removes embedded watermarks.

### EU AI Act Article 50

透明度 Code for AI-generated 内容 labeling (first draft December 2025, second draft March 2026, expected final June 2026 per the [European Commission status page](https://digital-strategy.ec.europa.eu/en/policies/code-practice-ai-generated-content)). The Code remains in draft as of April 2026 and the timeline is subject to change. The 监管 layer that requires the technical layer. Deepfakes must be labeled.

### Where this fits in Phase 18

Lessons 22-23 are about what the 模型 emits (private data, 来源证明 signal). Lesson 27 covers training-data 治理. Lesson 24 is the 监管 框架 that requires these technical measures.

## 使用它

`code/main.py` builds a toy text 水印. Tokens are integers 0..N-1; watermarked sampling biases toward the hash-defined green set. A detector computes the green-token z-score. You can observe detection at 1000-token generations, watch paraphrase destroy the signal, and measure the false-positive rate on human text.

## 交付它

本课产出 `outputs/skill-provenance-audit.md`. 给定 a 内容 部署 with a 来源证明 claim, it audits: the 水印 mechanism (if any), the C2PA signing chain (if any), the adversarial robustness of each, and the per-modality coverage.

## 练习

1. 运行 `code/main.py`. Report z-scores for watermarked 1000-token generation vs human-authored text. Identify the false-positive rate at the 95% confidence threshold.

2. Implement a paraphrase 攻击 that replaces 30% of tokens with synonyms. Re-measure the z-score.

3. 阅读 Kirchenbauer et al. 2023 Section 6 on robustness. Why do text watermarks fail under paraphrase but image watermarks survive cropping?

4. Design a 部署 that uses SynthID-text + C2PA metadata. 描述 the 来源证明 chain a consumer sees. Identify one failure mode of each component.

5. The 2024 "Stable Signature is Unstable" result shows 微调 removes the image 水印. Design a 部署 控制 that limits this 攻击 ， for example, require signed releases of fine-tuned checkpoints.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|-----------------|------------------------|
| SynthID | "Google's 水印" | Cross-modal 来源证明 signal; text, image, audio, video |
| Token 水印 | "Kirchenbauer-style" | Biased-sampling text 水印 detectable via green-token z-score |
| Stable Signature | "image 水印" | Fine-tuned-decoder 水印; ICCV 2023 |
| C2PA | "the metadata standard" | Cryptographically signed tamper-evident 来源证明 metadata |
| Paraphrase robustness | "does rewording break it" | Text 水印 property; currently limited |
| 微调 removal | "adversarial unwatermark" | 攻击 that removes image 水印 via decoder 微调 |
| Cross-modal detector | "unified SynthID" | November 2025 unified API across modalities |

## 延伸阅读

- [Kirchenbauer et al. ， A Watermark for Large Language Models (ICML 2023, arXiv:2301.10226)](https://arxiv.org/abs/2301.10226) ， the token-水印 mechanism
- [Fernandez et al. ， Stable Signature (ICCV 2023, arXiv:2303.15435)](https://arxiv.org/abs/2303.15435) ， image 水印 paper
- ["Stable Signature is Unstable" (arXiv:2405.07145)](https://arxiv.org/abs/2405.07145) ， the removal 攻击
- [Google DeepMind ， SynthID](https://deepmind.google/models/synthid/) ， the cross-modal 水印
- [C2PA 2.2 Explainer (2025)](https://c2pa.org/specifications/specifications/2.2/explainer/Explainer.html) ， metadata standard
