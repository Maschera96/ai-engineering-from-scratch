# ASCII Art and Visual Jailbreaks

> Jiang, Xu, Niu, Xiang, Ramasubramanian, Li, Poovendran, "ArtPrompt: ASCII Art-based 越狱 Attacks against Aligned LLMs" (ACL 2024, arXiv:2402.11753). Mask the 安全-relevant tokens in a harmful request, replace them with ASCII-art renderings of the same letters, and send the cloaked prompt. GPT-3.5, GPT-4, Gemini, Claude, Llama-2 all fail to robustly recognize ASCII-art tokens. The 攻击 bypasses PPL (perplexity filters), Paraphrase defenses, and Retokenization. Related: the ViTC 基准 measures recognition of non-semantic visual prompts; StructuralSleight generalizes to Uncommon Text-Encoded Structures (trees, graphs, nested JSON) as a family of encoding attacks.

**类型:** 构建
**语言:** Python (stdlib, ArtPrompt token-masking harness)
**先修要求:** Phase 18 · 12 (PAIR), Phase 18 · 13 (MSJ)
**时间:** ~60 分钟

## 学习目标

- 描述 the ArtPrompt 攻击: word-identification step, ASCII-art substitution, final cloaked prompt.
- 解释 why standard defenses (PPL, Paraphrase, Retokenization) fail on ArtPrompt.
- Define ViTC and describe what it measures.
- 描述 StructuralSleight as a generalization to arbitrary Uncommon Text-Encoded Structures.

## 问题

Attacks via paraphrase and roleplay (Lesson 12) and via long context (Lesson 13) operate on the text-level pattern. ArtPrompt operates at the recognition level: the 模型 does not parse the forbidden token. It parses an image rendered in characters. The 安全 filter sees harmless punctuation. The 模型 sees a word.

## 概念

### ArtPrompt, two steps

Step 1. Word Identification. 给定 a harmful request, the attacker uses an LLM to identify the 安全-relevant words (e.g., "bomb" in "how to make a bomb"). 

Step 2. Cloaked Prompt Generation. 替换 each identified word with its ASCII-art rendering (a 7x5 or 7x7 block of characters forming the letter shape). The 模型 receives a grid of punctuation and spaces that a sufficiently capable 模型 can recognize as the word; a 安全 filter sees only the grid.

Result: GPT-4, Gemini, Claude, Llama-2, GPT-3.5 all fail. 攻击 success rate above 75% on their 基准 subset.

### Why the standard defenses fail

- **PPL (perplexity filter).** ASCII art has high perplexity ， but so does all novel 输入. Threshold choices that block ArtPrompt also block legitimate structured 输入.
- **Paraphrase.** Paraphrasing the prompt destroys the ASCII art. In practice, paraphrase LLMs often preserve or reconstruct the art.
- **Retokenization.** Splitting tokens differently does not change that the 模型's vision is recognizing letter shapes.

The underlying issue is that 安全 filters are token- or semantic-level; ArtPrompt operates at the visual recognition level.

### ViTC 基准

Recognition of non-semantic visual prompts. Measures the 模型's ability to read ASCII-art, wingdings, and other non-text-semantic visual 内容. ArtPrompt's effectiveness correlates with ViTC accuracy: the better the 模型 reads visual text, the better ArtPrompt works on it. This is a 能力-安全 tradeoff.

### StructuralSleight

Generalizes ArtPrompt: Uncommon Text-Encoded Structures (UTES). Trees, graphs, nested JSON, CSV-in-JSON, diff-style code blocks. If a structure is rare in training 安全 data but parseable by the 模型, it can hide harmful 内容.

The 防御 implication: 安全 must generalize across the structured representations the 模型 can parse. The set is large and growing.

### Image-modality analog

Visual LLMs (GPT-5.2, Gemini 3 Pro, Claude Opus 4.5, Grok 4.1) extend the 攻击 surface. ArtPrompt-style attacks with actual images are stronger than ASCII-art analogs because image encoders produce richer signal.

### Where this fits in Phase 18

Lessons 12-14 describe three orthogonal 攻击 vectors: iterative refinement (PAIR), context length (MSJ), and encoding (ArtPrompt/StructuralSleight). Lesson 15 shifts from 模型-centric attacks to 系统-boundary attacks (indirect 提示注入). Lesson 16 describes the defensive tooling response.

## 使用它

`code/main.py` builds a toy ArtPrompt. You can cloak specific words in a harmful query with ASCII-art glyphs, verify the cloaked string passes a keyword filter, and (optionally) decode the cloaked string back using a simple recognizer.

## 交付它

本课产出 `outputs/skill-encoding-audit.md`. 给定 a 越狱-防御 report, it enumerates the encoding 攻击 families covered (ASCII art, base64, leet-speak, UTF-8 homoglyph, UTES) and the 防御 layer that catches each.

## 练习

1. 运行 `code/main.py`. Verify the cloaked string passes a simple keyword filter. Report the character-level change required.

2. Implement a second encoding: base64 for the same target word. 比较 the filter-bypass rate against ArtPrompt and the recovery difficulty.

3. 阅读 Jiang et al. 2024 Section 4.3 (five-模型 results). Propose a reason why Claude's ArtPrompt-resistance is higher than Gemini's on the same 基准.

4. Design a pre-generation 防御 that detects ASCII-art-shaped regions in the prompt. Measure the false-positive rate on legitimate code, tables, and mathematical notation.

5. StructuralSleight lists 10 encoding structures. Sketch a generalized 防御 that handles all 10 and estimate the compute cost per defended prompt.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|-----------------|------------------------|
| ArtPrompt | "the ASCII-art 攻击" | Two-step 越狱 that masks 安全 words with ASCII-art renderings |
| Cloaking | "hide the word" | 替换 a forbidden token with a visual representation the 模型 reads but the filter does not |
| UTES | "uncommon structure" | Uncommon Text-Encoded Structure ， tree, graph, nested JSON, etc. used to smuggle 内容 |
| ViTC | "visual-text 能力" | 基准 for 模型's ability to read non-semantic visual encoding |
| Perplexity filter | "PPL 防御" | Reject prompts with high perplexity; fails because legitimate structured 输入 also scores high |
| Retokenization | "tokenizer shift 防御" | Pre-process the prompt with a different tokenizer; fails because recognition is visual |
| Homoglyph | "lookalike characters" | Unicode characters that look identical to Latin letters; bypass substring checks |

## 延伸阅读

- [Jiang et al. ， ArtPrompt (ACL 2024, arXiv:2402.11753)](https://arxiv.org/abs/2402.11753) ， the ASCII-art 越狱 paper
- [Li et al. ， StructuralSleight (arXiv:2406.08754)](https://arxiv.org/abs/2406.08754) ， UTES generalization
- [Chao et al. ， PAIR (Lesson 12, arXiv:2310.08419)](https://arxiv.org/abs/2310.08419) ， complementary iterative 攻击
- [Anil et al. ， Many-shot Jailbreaking (Lesson 13)](https://www.anthropic.com/research/many-shot-jailbreaking) ， complementary length 攻击
