# 审核 Systems ， OpenAI, Perspective, Llama Guard

> Production 审核 systems operationalize the 安全 policies defined in Lessons 12-16. OpenAI 审核 API: `omni-moderation-latest` (2024) built on GPT-4o classifies text + images in one call; 42% better on multilingual test set than prior version; the response schema returns 13 category booleans ， harassment, harassment/threatening, hate, hate/threatening, illicit, illicit/violent, self-harm, self-harm/intent, self-harm/instructions, sexual, sexual/minors, violence, violence/graphic; free for most developers. Layered patterns: 输入 审核 (pre-generation), 输出 审核 (post-generation), Custom 审核 (domain rules). Async parallel calls hide latency; placeholder responses on flag. Llama Guard 3/4 (Lesson 16): 14 MLCommons hazards, Code Interpreter Abuse, 8 languages (v3), multi-image (v4). Perspective API (Google Jigsaw): toxicity scoring predating the LLM-as-moderator wave; primarily single-dimension toxicity with severe-toxicity/insult/profanity variants; baseline for 内容-审核 research. Deprecations: Azure 内容 Moderator deprecated February 2024, retired February 2027, replaced by Azure AI 内容 安全.

**类型:** 构建
**语言:** Python (stdlib, three-layer 审核 harness)
**先修要求:** Phase 18 · 16 (Llama Guard / Garak / PyRIT)
**时间:** ~60 分钟

## 学习目标

- 描述 the OpenAI 审核 API's category taxonomy and how it differs from Llama Guard 3's MLCommons set.
- 描述 the three 审核-layer pattern (输入, 输出, custom) and name one failure mode of each.
- 描述 Perspective API's position as a pre-LLM-era baseline and why it remains used in research.
- 说明 the Azure deprecation timeline.

## 问题

Lessons 12-16 describe attacks and 防御 tooling. Lesson 29 covers the deployed 审核 systems that operationalize the defenses at the surface where users touch the product. The three-layer pattern is the 2026 default configuration.

## 概念

### OpenAI 审核 API

`omni-moderation-latest` (2024). Built on GPT-4o. Classifies text + images in one call. Free for most developers.

Categories (13 booleans in the response schema):
- harassment, harassment/threatening
- hate, hate/threatening
- self-harm, self-harm/intent, self-harm/instructions
- sexual, sexual/minors
- violence, violence/graphic
- illicit, illicit/violent

Multimodal support applies to `violence`, `self-harm`, and `sexual` but not `sexual/minors`; the rest are text-only.

For the code harness in `code/main.py` we collapse the `/threatening`, `/intent`, `/instructions`, and `/graphic` sub-categories into their top-level parents for pedagogical simplicity. Production code should use the full 13-category schema.

42% better on multilingual test set than the prior-generation 审核 endpoint. Per-category scores; applications set thresholds.

### Llama Guard 3/4

Covered in Lesson 16. 14 MLCommons hazard categories (organized differently from OpenAI's 13 response-schema booleans). Supports 8 languages (v3). Llama Guard 4 (April 2025) is natively multimodal, 12B.

The OpenAI and Llama Guard taxonomies overlap but diverge. OpenAI has "illicit" as a broad category; Llama Guard has "violent crimes" and "non-violent crimes" separately. Deployments pick based on their 策略-taxonomy fit.

### Perspective API (Google Jigsaw)

Toxicity scoring 系统 predating the LLM-as-moderator wave (pre-2020). Categories: TOXICITY, SEVERE_TOXICITY, INSULT, PROFANITY, THREAT, IDENTITY_ATTACK. Single-dimension primary score (TOXICITY) with sub-dimension variants.

Widely used as a 内容-审核 research baseline because the API is stable, documented, and has years of calibration data. For modern LLM-adjacent use cases, Llama Guard or OpenAI 审核 is typically a better fit.

### The three-layer pattern

1. **输入 审核.** Classify the user's prompt before generation. Reject if flagged. Latency: one 分类器 call.
2. **输出 审核.** Classify the 模型's 输出 before delivery. 替换 with a refusal if flagged. Latency: one 分类器 call after generation.
3. **Custom 审核.** Domain-specific rules (regex, allowlists, business 策略). Runs at either 输入 or 输出.

The three layers are sequential by design: 输入 审核 must complete before generation, and 输出 审核 runs after generation. Parallelism applies within a layer ， running multiple classifiers (e.g., OpenAI 审核 + Llama Guard + Perspective) concurrently on the same text hides per-分类器 latency. As an optional optimization, a placeholder response ("one moment, checking...") may be shown while 输入 审核 completes and token-1 streaming is deferred. Flag behaviour is configurable: refuse, sanitize, escalate to human review.

### Failure modes

- **输入 only.** Does not catch 输出 hallucinations (Lesson 12-14 encoding attacks bypass 输入 classifiers).
- **输出 only.** Allows any 输入 to reach the 模型; increases cost; surfaces internal reasoning to attacker.
- **Custom only.** Not robust across categories; regexes are brittle.

Layered is the default. Belt-and-suspenders.

### Azure deprecation

Azure 内容 Moderator: deprecated February 2024, retired February 2027. Replaced by Azure AI 内容 安全, which is LLM-based and integrates with Azure OpenAI. The migration is a 2024-2027 field-level project for Azure deployments.

### Where this fits in Phase 18

Lesson 16 covers the 审核 tooling in the 红队 context. Lesson 29 covers deployed 审核. Lesson 30 closes with the current 双重用途 能力 evidence.

## 使用它

`code/main.py` builds a three-layer 审核 harness: 输入 moderator (keyword + category score), 输出 moderator (same 分类器 on 输出), custom moderator (domain rules). You can run inputs through and observe which layer catches what.

## 交付它

本课产出 `outputs/skill-moderation-stack.md`. 给定 a 部署, it recommends a 审核 stack configuration: which 分类器 at 输入, which at 输出, which custom rules, and what judge for edge cases.

## 练习

1. 运行 `code/main.py`. 运行 a benign, borderline, and harmful 输入 through all three layers. Report which layer fires for each.

2. Extend the harness with Perspective-API-style toxicity scoring on a specific category. 比较 its threshold behaviour to the category score.

3. 阅读 the OpenAI 审核 API docs and the Llama Guard 3 category list. Map each OpenAI category to the closest Llama Guard categories. Identify three categories that do not cleanly map.

4. Design a 审核 stack for a code-assistant 部署 (e.g., GitHub Copilot). Identify the categories most and least relevant and propose custom rules.

5. Azure 内容 Moderator retires February 2027. Plan a migration to Azure AI 内容 安全. Identify the highest-风险 element of the migration.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|-----------------|------------------------|
| OpenAI 审核 | "omni-审核-latest" | GPT-4o-based 13-category (text) 分类器 with partial multimodal support |
| Perspective API | "Google Jigsaw toxicity" | Pre-LLM-era toxicity scoring baseline |
| Llama Guard | "MLCommons 14-category" | Meta's hazard 分类器 (v3: 8B text, 8 langs; v4: 12B multimodal) |
| 输入 审核 | "pre-generation filter" | 分类器 on user prompt before 模型 call |
| 输出 审核 | "post-generation filter" | 分类器 on 模型 输出 before delivery |
| Custom 审核 | "domain rules" | 部署-specific rules (regex, allowlist, 策略) |
| Layered 审核 | "all three layers" | Standard production 部署 pattern |

## 延伸阅读

- [OpenAI Moderation API docs](https://platform.openai.com/docs/api-reference/moderations) ， omni-审核 endpoint
- [Meta PurpleLlama + Llama Guard](https://github.com/meta-llama/PurpleLlama) ， Llama Guard repo
- [Google Jigsaw Perspective API](https://perspectiveapi.com/) ， toxicity scoring
- [Azure AI Content Safety](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/) ， Azure replacement
