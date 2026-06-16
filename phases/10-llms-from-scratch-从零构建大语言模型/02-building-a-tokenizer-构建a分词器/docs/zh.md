# Building a 分词器 from Scratch

> Lesson 01 gave you a toy. This lesson gives you a weapon.

**类型：** Build
**语言：** Python
**先修：** Phase 10, Lesson 01 (分词器s: BPE, WordPiece, SentencePiece)
**时间：** 约 90 分钟

## 学习目标

- 构建a production-grade BPE 分词器 that handles Unicode, whitespace 归一化, and special 词元
- Implement byte-level 备选方案 so the 分词器 can encode any 输入 (including emoji, CJK, and code) without unknown 词元
- Add pre-tokenization regex patterns that split 文本 at word boundaries before applying BPE merges
- 训练 a custom 分词器 on a 语料库 and evaluate its 压缩 比例 against tiktoken on multilingual 文本

## 问题

你的BPE 分词器 from Lesson 01 works on English 文本. Now throw Japanese at it. Or emoji. Or Python code with mixed tabs and spaces.

It breaks.

Not because BPE is wrong -- because the implementation is incomplete. A 生产 分词器 handles raw bytes in any encoding, normalizes Unicode before splitting, manages special 词元 that never get merged, chains pre-tokenization with subword splitting, and does all of this fast enough to not bottleneck a 训练 流水线 processing 15 trillion 词元.

GPT-2's 分词器 has 50,257 词元. Llama 3 has 128,256. GPT-4 has roughly 100,000. These are not toy numbers. The merge tables behind those vocabularies were 训练后的 on hundreds of gigabytes of 文本, and the surrounding machinery -- 归一化, pre-tokenization, special 词元 injection, chat template formatting -- is what separates a 分词器 that handles "hello world" from one that handles the entire internet.

你are going to build that machinery.

## 概念

### The Full 流水线

一个生产 分词器 is not one algorithm. It is a 流水线 of five stages, each solving a different problem.

```mermaid
graph LR
    A[Raw 文本] --> B[Normalize]
    B --> C[Pre-Tokenize]
    C --> D[BPE Merge]
    D --> E[Special 词元]
    E --> F[词元 IDs]

    style A fill:#1a1a2e,stroke:#e94560,color:#fff
    style B fill:#1a1a2e,stroke:#e94560,color:#fff
    style C fill:#1a1a2e,stroke:#e94560,color:#fff
    style D fill:#1a1a2e,stroke:#e94560,color:#fff
    style E fill:#1a1a2e,stroke:#e94560,color:#fff
    style F fill:#1a1a2e,stroke:#e94560,color:#fff
```

Each stage has a specific job:

|Stage|What It Does|Why It Matters|
|-------|-------------|----------------|
|Normalize|NFKC Unicode, lowercase optional, strip accents optional|"fi" ligature (U+FB01) becomes "fi" (two chars). Without this, same word gets different 词元.|
|Pre-Tokenize|Split 文本 into chunks before BPE|Prevents BPE from merging across word boundaries. "the cat" should never produce a 词元 "e c".|
|BPE Merge|Apply learned merge rules to byte sequences|The core 压缩. Turns raw bytes into subword 词元.|
|Special 词元|Inject [BOS], [EOS], [PAD], chat template markers|These 词元 have fixed IDs. They never participate in BPE merges. The 模型 needs them for structure.|
|ID Mapping|Convert 词元 strings to integer IDs|The 模型 sees integers, not strings.|

### Byte-Level BPE

Lesson 01's 分词器 operated on UTF-8 bytes. That was the right call. But we skipped something important: what happens when those bytes are not 有效 UTF-8?

Byte-level BPE solves this by treating every possible byte value (0-255) as a 有效 词元. Your base 词表 is exactly 256 entries. Any file -- 文本, binary, corrupted -- can be tokenized without producing an unknown 词元.

GPT-2 added a trick: map each byte to a printable Unicode character so the 词表 stays human-readable. Byte 0x20 (space) becomes the character "G" in their mapping. This is purely cosmetic. The algorithm does not care.

这个真实 power: byte-level BPE handles every language on earth. Chinese characters are 3 UTF-8 bytes each. Japanese can be 3-4 bytes. Arabic, Devanagari, emoji -- all just byte sequences. The BPE algorithm finds patterns in these byte sequences exactly the same way it finds patterns in English ASCII bytes.

### Pre-Tokenization

Before BPE touches your 文本, you need to split it into chunks. This prevents the merge algorithm from creating 词元 that span word boundaries.

GPT-2 uses a regex pattern to split 文本:

```text
'(?:[sdmt]|ll|ve|re)| ?\p{L}+| ?\p{N}+| ?[^\s\p{L}\p{N}]+|\s+(?!\S)|\s+
```

这pattern splits on contractions ("don't" becomes "don" + "'t"), words with optional leading spaces, numbers, punctuation, and whitespace. The leading space is kept attached to the word -- so "the cat" becomes [" the", " cat"], not ["the", " ", "cat"].

Llama uses SentencePiece, which skips regex entirely. It treats the raw byte stream as one long 序列 and lets the BPE algorithm figure out the boundaries. This is simpler but gives BPE more freedom to create cross-word 词元.

这个choice matters. GPT-2's regex prevents the 分词器 from 学习 that "the" at the end of one word and "the" at the start of the next should merge. SentencePiece allows it, which sometimes produces more efficient 压缩 but less interpretable 词元.

### Special 词元

每个生产 分词器 reserves 词元 IDs for structural markers:

|词元|Purpose|Used By|
|-------|---------|---------|
|`[BOS]` / `<s>`|Beginning of 序列|Llama 3, GPT|
|`[EOS]` / `</s>`|End of 序列|All 模型|
|`[PAD]`|Padding for 批次 对齐|BERT, T5|
|`[UNK]`|Unknown 词元 (byte-level BPE eliminates this)|BERT, WordPiece|
|`<\|im_start\|>`|Chat 消息 boundary start|ChatGPT, Qwen|
|`<\|im_end\|>`|Chat 消息 boundary end|ChatGPT, Qwen|
|`<\|用户\|>`|用户 turn marker|Llama 3|
|`<\|助手\|>`|助手 turn marker|Llama 3|

Special 词元 are never split by BPE. They are matched exactly before the merge algorithm runs, replaced with their fixed ID, and the surrounding 文本 is tokenized normally.

### Chat Templates

这is where most people get confused and most implementations break.

当you send 消息 to a chat 模型, the API accepts a list of 消息:

```text
[
  {"role": "system", "content": "You are helpful."},
  {"role": "user", "content": "Hello"},
  {"role": "assistant", "content": "Hi there!"}
]
```

这个模型 does not see JSON. It sees a flat 词元 序列. The chat template converts 消息 into that flat 序列 using special 词元. Every 模型 does this differently:

```text
Llama 3:
<|begin_of_text|><|start_header_id|>system<|end_header_id|>

You are helpful.<|eot_id|><|start_header_id|>user<|end_header_id|>

Hello<|eot_id|><|start_header_id|>assistant<|end_header_id|>

Hi there!<|eot_id|>

ChatGPT:
<|im_start|>system
You are helpful.<|im_end|>
<|im_start|>user
Hello<|im_end|>
<|im_start|>assistant
Hi there!<|im_end|>
```

Get the template wrong and the 模型 produces garbage. It was 训练后的 on one exact format. Any deviation -- a missing newline, a swapped 词元, an extra space -- puts the 输入 outside the 训练 分布.

### Speed

Python is too slow for 生产 tokenization.

tiktoken (OpenAI) is written in Rust with Python bindings. HuggingFace 分词器s is also Rust. SentencePiece is C++. These achieve 10-100x speedups over pure Python.

For perspective: tokenizing 15 trillion 词元 for Llama 3 预训练 at 1 million 词元 per second (fast Python) would take 174 days. At 100 million 词元 per second (Rust), it takes 1.7 days.

你are building in Python to understand the algorithm. In 生产, you would use a compiled implementation and only touch the Python wrapper.

```figure
weight-tying
```

## 动手构建

### 步骤 1: Byte-Level Encoding

这个foundation. Convert any string into a 序列 of bytes, map each byte to a printable character for display, and reverse the process.

```python
def bytes_to_tokens(text):
    return list(text.encode("utf-8"))

def tokens_to_text(token_bytes):
    return bytes(token_bytes).decode("utf-8", errors="replace")
```

Test on multilingual 文本 to see the byte counts:

```python
texts = [
    ("English", "hello"),
    ("Chinese", "你好"),
    ("Emoji", "🔥"),
    ("Mixed", "hello你好🔥"),
]

for label, text in texts:
    b = bytes_to_tokens(text)
    print(f"{label}: {len(text)} chars -> {len(b)} bytes -> {b}")
```

"hello" is 5 bytes. "你好" is 6 bytes (3 per character). The fire emoji is 4 bytes. The byte-level 分词器 does not care what language it is. Bytes are bytes.

### 步骤 2: Pre-分词器 with Regex

Split 文本 into chunks using the GPT-2 regex pattern. Each 分块 gets tokenized independently by BPE.

```python
import re

try:
    import regex
    GPT2_PATTERN = regex.compile(
        r"""'(?:[sdmt]|ll|ve|re)| ?\p{L}+| ?\p{N}+| ?[^\s\p{L}\p{N}]+|\s+(?!\S)|\s+"""
    )
except ImportError:
    GPT2_PATTERN = re.compile(
        r"""'(?:[sdmt]|ll|ve|re)| ?[a-zA-Z]+| ?[0-9]+| ?[^\s\w]+|\s+(?!\S)|\s+"""
    )

def pre_tokenize(text):
    return [match.group() for match in GPT2_PATTERN.finditer(text)]
```

这个`regex` module supports Unicode property escapes (`\p{L}` for letters, `\p{N}` for numbers). The standard library `re` module does not, so we fall back to ASCII character classes. For 生产 multilingual 分词器s, install `regex`.

Try it:

```python
print(pre_tokenize("Hello, world! Don't stop."))
# [' Hello', ',', ' world', '!', " Don", "'t", ' stop', '.']
```

这个leading space stays attached to the word. Contractions split at the apostrophe. Punctuation becomes its own 分块. BPE will never merge 词元 across these boundaries.

### 步骤 3: BPE on Byte Sequences

这个core algorithm from Lesson 01, but now operating on pre-tokenized chunks independently.

```python
from collections import Counter

def get_byte_pairs(chunks):
    pairs = Counter()
    for chunk in chunks:
        byte_seq = list(chunk.encode("utf-8"))
        for i in range(len(byte_seq) - 1):
            pairs[(byte_seq[i], byte_seq[i + 1])] += 1
    return pairs

def apply_merge(byte_seq, pair, new_id):
    merged = []
    i = 0
    while i < len(byte_seq):
        if i < len(byte_seq) - 1 and byte_seq[i] == pair[0] and byte_seq[i + 1] == pair[1]:
            merged.append(new_id)
            i += 2
        else:
            merged.append(byte_seq[i])
            i += 1
    return merged
```

### 步骤 4: Special 词元 Handling

Special 词元 need exact 匹配 and fixed IDs. They bypass BPE entirely.

```python
class SpecialTokenHandler:
    def __init__(self):
        self.special_tokens = {}
        self.pattern = None

    def add_token(self, token_str, token_id):
        self.special_tokens[token_str] = token_id
        escaped = [re.escape(t) for t in sorted(self.special_tokens.keys(), key=len, reverse=True)]
        self.pattern = re.compile("|".join(escaped))

    def split_with_specials(self, text):
        if not self.pattern:
            return [(text, False)]
        parts = []
        last_end = 0
        for match in self.pattern.finditer(text):
            if match.start() > last_end:
                parts.append((text[last_end:match.start()], False))
            parts.append((match.group(), True))
            last_end = match.end()
        if last_end < len(text):
            parts.append((text[last_end:], False))
        return parts
```

### 步骤 5: Full 分词器 Class

链 everything together: normalize, split on special 词元, pre-tokenize, BPE merge, map to IDs.

```python
import unicodedata

class ProductionTokenizer:
    def __init__(self):
        self.merges = {}
        self.vocab = {i: bytes([i]) for i in range(256)}
        self.special_handler = SpecialTokenHandler()
        self.next_id = 256

    def normalize(self, text):
        return unicodedata.normalize("NFKC", text)

    def train(self, text, num_merges):
        text = self.normalize(text)
        chunks = pre_tokenize(text)
        chunk_bytes = [list(chunk.encode("utf-8")) for chunk in chunks]

        for i in range(num_merges):
            pairs = Counter()
            for seq in chunk_bytes:
                for j in range(len(seq) - 1):
                    pairs[(seq[j], seq[j + 1])] += 1
            if not pairs:
                break
            best = max(pairs, key=pairs.get)
            new_id = self.next_id
            self.next_id += 1
            self.merges[best] = new_id
            self.vocab[new_id] = self.vocab[best[0]] + self.vocab[best[1]]
            chunk_bytes = [apply_merge(seq, best, new_id) for seq in chunk_bytes]

    def add_special_token(self, token_str):
        token_id = self.next_id
        self.next_id += 1
        self.special_handler.add_token(token_str, token_id)
        self.vocab[token_id] = token_str.encode("utf-8")
        return token_id

    def encode(self, text):
        text = self.normalize(text)
        parts = self.special_handler.split_with_specials(text)
        all_ids = []
        for part_text, is_special in parts:
            if is_special:
                all_ids.append(self.special_handler.special_tokens[part_text])
            else:
                for chunk in pre_tokenize(part_text):
                    byte_seq = list(chunk.encode("utf-8"))
                    for pair, new_id in self.merges.items():
                        byte_seq = apply_merge(byte_seq, pair, new_id)
                    all_ids.extend(byte_seq)
        return all_ids

    def decode(self, ids):
        byte_parts = []
        for token_id in ids:
            if token_id in self.vocab:
                byte_parts.append(self.vocab[token_id])
        return b"".join(byte_parts).decode("utf-8", errors="replace")

    def vocab_size(self):
        return len(self.vocab)
```

### 步骤 6: Multilingual Test

这个真实 test. Throw English, Chinese, emoji, and code at it.

```python
corpus = (
    "The quick brown fox jumps over the lazy dog. "
    "The quick brown fox runs through the forest. "
    "Machine learning models process natural language. "
    "Deep learning transforms how we build software. "
    "def train(model, data): return model.fit(data) "
    "def predict(model, x): return model(x) "
)

tok = ProductionTokenizer()
tok.train(corpus, num_merges=50)

bos = tok.add_special_token("<|begin|>")
eos = tok.add_special_token("<|end|>")

test_texts = [
    "The quick brown fox.",
    "你好世界",
    "Hello 🌍 World",
    "def foo(x): return x + 1",
    f"<|begin|>Hello<|end|>",
]

for text in test_texts:
    ids = tok.encode(text)
    decoded = tok.decode(ids)
    print(f"Input:   {text}")
    print(f"Tokens:  {len(ids)} ids")
    print(f"Decoded: {decoded}")
    print()
```

Chinese characters produce 3 bytes each. The emoji produces 4 bytes. None of these crash the 分词器. None produce unknown 词元. That is the power of byte-level BPE.

## 实际使用

### Comparing 真实 分词器s

Load the actual 分词器s from Llama 3, GPT-4, and Mistral. See how each handles the same multilingual paragraph.

```python
import tiktoken

gpt4_enc = tiktoken.get_encoding("cl100k_base")

test_paragraph = "Machine learning is powerful. 机器学习很强大。 L'apprentissage automatique est puissant. 🤖💪"

tokens = gpt4_enc.encode(test_paragraph)
pieces = [gpt4_enc.decode([t]) for t in tokens]
print(f"GPT-4 ({len(tokens)} tokens): {pieces}")
```

```python
from transformers import AutoTokenizer

llama_tok = AutoTokenizer.from_pretrained("meta-llama/Meta-Llama-3-8B")
mistral_tok = AutoTokenizer.from_pretrained("mistralai/Mistral-7B-v0.1")

for name, tok in [("Llama 3", llama_tok), ("Mistral", mistral_tok)]:
    tokens = tok.encode(test_paragraph)
    pieces = tok.convert_ids_to_tokens(tokens)
    print(f"{name} ({len(tokens)} tokens): {pieces[:20]}...")
```

你will see different 词元 counts for the same 文本. Llama 3 with 128K 词表 is more aggressive at merging common patterns. GPT-4 with 100K sits in the middle. Mistral with 32K produces more 词元 but has a smaller 嵌入 层.

这个tradeoff is always the same: larger 词表 means shorter sequences but more 参数.

## 交付成果

这lesson produces a 提示词 for building and 调试 生产 分词器s. See `outputs/prompt-tokenizer-builder.md`.

## 练习

1. **Easy:** Add a `get_token_bytes(id)` method that shows the raw bytes for any 词元 ID. Use it to inspect what your most common merged 词元 actually represent.
2. **Medium:** Implement the Llama-style pre-分词器 that splits on whitespace and digits but keeps leading spaces. Compare its 词表 with the GPT-2 regex approach on the same 语料库.
3. **Hard:** Add a chat template method that takes a list of `{"role": ..., "content": ...}` 消息 and produces the correct 词元 序列 for the Llama 3 chat format. Test it against the HuggingFace implementation.

## Key Terms

|Term|What people say|What it actually means|
|------|----------------|----------------------|
|Byte-level BPE|"分词器 that works on bytes"|BPE with a base 词表 of 256 byte values -- handles any 输入 without unknown 词元|
|Pre-tokenization|"Splitting before BPE"|Regex or rule-based splitting that prevents BPE from merging across word boundaries|
|NFKC 归一化|"Unicode cleanup"|Canonical decomposition followed by compatibility composition -- "fi" ligature becomes "fi", fullwidth "A" becomes "A"|
|Chat template|"How 消息 become 词元"|The exact format for converting a list of role/content 消息 into a flat 词元 序列 -- model-specific and must match 训练 format|
|Special 词元|"Control 词元"|Reserved 词元 IDs that bypass BPE -- [BOS], [EOS], [PAD], chat markers -- matched exactly before merge|
|Fertility|"词元 per word"|比例 of 输出 词元 to 输入 words -- 1.3 for English in GPT-4, 2-3 for Korean, higher means wasted 上下文|
|tiktoken|"OpenAI 分词器"|Rust BPE implementation with Python bindings -- 10-100x faster than pure Python|
|Merge table|"The 词表"|Ordered list of byte-pair merges learned during 训练 -- this IS the 分词器's learned knowledge|

## 延伸阅读

- [OpenAI tiktoken source](https://github.com/openai/tiktoken) -- Rust BPE implementation used by GPT-3.5/4
- [HuggingFace tokenizers](https://github.com/huggingface/tokenizers) -- Rust 分词器 library supporting BPE, WordPiece, Unigram
- [Llama 3 paper (Meta, 2024)](https://arxiv.org/abs/2407.21783) -- details on 128K 词表 and 分词器 训练
- [SentencePiece (Kudo & Richardson, 2018)](https://arxiv.org/abs/1808.06226) -- language-agnostic tokenization
- [GPT-2 tokenizer source](https://github.com/openai/gpt-2/blob/master/src/encoder.py) -- the original byte-to-Unicode mapping
