# 注意力机制，关键突破

> 这个 decoder stops squinting at a compressed 摘要 与 starts looking at the whole source. Everything after this is 注意力 plus engineering.

**类型:** 构建
**语言:** Python
**先修要求:** Phase 5 · 09 (序列-to-序列 Models)
**时间:** 约45分钟

## 问题

Lesson 09 ended on a measured failure. A GRU encoder-decoder trained on a toy copy 任务 goes 从 89% 准确率 at length 5 to near-chance at length 80. The 原因 is structural, not a 训练 bug: every bit of information the encoder gleaned has to fit in one 固定-size hidden 状态, 与 the decoder never sees anything else.

Bahdanau, Cho, 与 Bengio published a three-line fix in 2014. Instead of giving the decoder only the final encoder 状态, keep every encoder 状态. At each decoder step, 计算 a weighted average of encoder states 其中 the weights say "how much does the decoder need to look at encoder position `i` right now?" 这 weighted average is the context, 与 it changes every decoder step.

这 is the whole idea. Transformers extended it. Self-注意力 applied it to a single 序列. Multi-head 注意力 ran it in parallel. But the 2014 version already broke the bottleneck, 与 once you have it, the pivot to transformers is engineering, not conceptual.

## 概念

![说明：Bahdanau 注意力: decoder queries all encoder states](../assets/attention.svg)

At each decoder step `t`:

1. Use the previous decoder hidden 状态 `s_{t-1}` as a **查询**.
2. 说明：Score it against every encoder hidden 状态 `h_1, ..., h_T`. One scalar per encoder position.
3. Softmax the scores to get 注意力 weights `α_{t,1}, ..., α_{t,T}` 这 sum to 1.
4. 说明：Context 向量 `c_t = Σ α_{t,i} * h_i`. Weighted average of encoder states.
5. 说明：Decoder takes `c_t` plus the previous 输出 词元, produces the next 词元.

说明：这个 weighted average is the point. When the decoder needs to translate "Je" to "I", it weights the encoder 状态 over "Je" high 与 the others low. When it needs "not", it weights "pas" high. The context 向量 reshapes each step.

## Shapes (the thing 这 bites everyone)

说明：This is 其中 every 注意力 implementation goes 错误 the first time. Read slowly.

|Thing|Shape|Notes|
|-------|-------|-------|
|Encoder hidden states `H`|`(T_enc, d_h)`|如果 BiLSTM, `d_h = 2 * d_hidden`|
|Decoder hidden 状态 `s_{t-1}`|`(d_s,)`|One 向量|
|注意力 score `e_{t,i}`|scalar|One per encoder position|
|注意力 weight `α_{t,i}`|scalar|After softmax over all `i`|
|Context 向量 `c_t`|`(d_h,)`|Same shape as an encoder 状态|

**Bahdanau (additive) score.** `e_{t,i} = v_α^T * tanh(W_a * s_{t-1} + U_a * h_i)`.

- `s_{t-1}` has shape `(d_s,)`, `h_i` has shape `(d_h,)`.
- `W_a` has shape `(d_attn, d_s)`. `U_a` has shape `(d_attn, d_h)`.
- Their sum inside the tanh has shape `(d_attn,)`.
- `v_α` has shape `(d_attn,)`. The inner product 使用 `v_α` collapses to a scalar. **This is what `v_α` does.** It is not magic. It is the projection 这 turns an 注意力-dim 向量 到 a scalar score.

说明：**Luong (multiplicative) score.** Three variants:

- 说明：`dot`: `e_{t,i} = s_t^T * h_i`. Requires `d_s == d_h`. Hard constraint. Skip if your encoder is bidirectional.
- 说明：`general`: `e_{t,i} = s_t^T * W * h_i` 使用 `W` shape `(d_s, d_h)`. Removes the equal-dim constraint.
- 说明：`concat`: essentially the Bahdanau form. Rarely used since the first two are cheaper.

**One Bahdanau / Luong gotcha worth naming.** Bahdanau uses `s_{t-1}` (the decoder 状态 *before* generating the 当前 词). Luong uses `s_t` (the 状态 *after*). Mixing them up produces subtly 错误 gradients 这 are extremely hard to debug. Pick one paper 与 stick to its convention.

```figure
attention-heatmap
```

## 动手构建

### Step 1: additive (Bahdanau) 注意力

```python
import numpy as np


def additive_attention(decoder_state, encoder_states, W_a, U_a, v_a):
    projected_dec = W_a @ decoder_state
    projected_enc = encoder_states @ U_a.T
    combined = np.tanh(projected_enc + projected_dec)
    scores = combined @ v_a
    weights = softmax(scores)
    context = weights @ encoder_states
    return context, weights


def softmax(x):
    x = x - np.max(x)
    e = np.exp(x)
    return e / e.sum()
```

说明：Check your shapes against the table above. `encoder_states` has shape `(T_enc, d_h)`. `projected_enc` has shape `(T_enc, d_attn)`. `projected_dec` has shape `(d_attn,)` 与 broadcasts. `combined` has shape `(T_enc, d_attn)`. `scores` has shape `(T_enc,)`. `weights` has shape `(T_enc,)`. `context` has shape `(d_h,)`. Ship it.

### Step 2: Luong dot 与 general

```python
def dot_attention(decoder_state, encoder_states):
    scores = encoder_states @ decoder_state
    weights = softmax(scores)
    return weights @ encoder_states, weights


def general_attention(decoder_state, encoder_states, W):
    projected = W.T @ decoder_state
    scores = encoder_states @ projected
    weights = softmax(scores)
    return weights @ encoder_states, weights
```

说明：Three lines each. This is why Luong's paper landed. Same 准确率 on most tasks, a lot less code.

### Step 3: a worked numerical example

Given three encoder states (roughly "cat", "sat", "mat") 与 a decoder 状态 这 aligns most 使用 the first, the 注意力 分布 concentrates on position 0. If the decoder 状态 shifts to align 使用 the last, 注意力 moves to position 2. The context 向量 tracks.

```python
H = np.array([
    [1.0, 0.0, 0.2],
    [0.5, 0.5, 0.1],
    [0.1, 0.9, 0.3],
])

s_close_to_cat = np.array([0.9, 0.1, 0.2])
ctx, w = dot_attention(s_close_to_cat, H)
print("weights:", w.round(3))
```

```
weights: [0.464 0.305 0.231]
```

First row wins. Then move the decoder 状态 closer to the third encoder 状态 与 watch the weights shift. 这 is it. 注意力 is explicit alignment.

### 说明：Step 4: why this is the bridge to transformers

Translate the language above 到 Q/K/V:

- **Query** = decoder 状态 `s_{t-1}`
- **Key** = encoder states (what we score against)
- **值** = encoder states (what we weight 与 sum)

在 classical 注意力, keys 与 values are the same thing. Self-注意力 separates them: you can 查询 a 序列 against itself, 使用 different learned projections 面向 K 与 V. Multi-head 注意力 runs it in parallel 使用 different learned projections. Transformers stack the whole stage many times 与 drop RNNs.

这个 math is the same. The shapes are the same. The pedagogical jump 从 Bahdanau 注意力 to scaled dot-product 注意力 is mostly notation.

## 投入使用

PyTorch 与 TensorFlow ship 注意力 directly.

```python
import torch
import torch.nn as nn

mha = nn.MultiheadAttention(embed_dim=128, num_heads=8, batch_first=True)
query = torch.randn(2, 5, 128)
key = torch.randn(2, 10, 128)
value = torch.randn(2, 10, 128)

output, weights = mha(query, key, value)
print(output.shape, weights.shape)
```

```
torch.Size([2, 5, 128]) torch.Size([2, 5, 10])
```

说明：这 is a transformer 注意力 layer. Query batch of 5 positions, key/值 batch of 10 positions, 128-dim each, 8 heads. `output` is the new context-augmented queries. `weights` is the 5x10 alignment 矩阵 you can visualize.

### 当 classical 注意力 still matters

- 说明：Pedagogy. The single-head, single-layer, RNN-based version makes every concept visible.
- On-device 序列 tasks 其中 transformers do not fit.
- 说明：Any paper 从 2014-2017. You will misread it 不使用 knowing Bahdanau's convention.
- 说明：Fine-grained alignment analysis in MT. 原始 注意力 weights are an interpretability tool even on transformer models, 与 reading them requires knowing what they are.

### 这个 注意力-weight-as-explanation trap

说明：注意力 weights look interpretable. They are weights 这 sum to one across positions; you can plot them; high means "looked at this." Reviewers love them.

They are not as interpretable as they look. Jain 与 Wallace (2019) showed 这 注意力 distributions can be permuted 与 replaced by arbitrary alternatives 不使用 changing 模型 predictions 面向 some tasks. Never report 注意力 weights as evidence of reasoning 不使用 an ablation 或 counterfactual check.

## 交付成果

保存为 `outputs/prompt-attention-shapes.md`:

```markdown
---
name: attention-shapes
description: Debug shape bugs in attention implementations.
phase: 5
lesson: 10
---

Given a broken attention implementation, you identify the shape mismatch. Output:

1. Which matrix has the wrong shape. Name the tensor.
2. What its shape should be, derived from (d_s, d_h, d_attn, T_enc, T_dec, batch_size).
3. One-line fix. Transpose, reshape, or project.
4. A test to catch regressions. Typically: assert `output.shape == (batch, T_dec, d_h)` and `weights.shape == (batch, T_dec, T_enc)` and `weights.sum(dim=-1) close to 1`.

Refuse to recommend fixes that silently broadcast. Broadcast-hiding bugs surface later as silent accuracy degradation, the worst kind of attention bug.

For Bahdanau confusion, insist the decoder input is `s_{t-1}` (pre-step state). For Luong, `s_t` (post-step state). For dot-product, flag dimension mismatch between query and key as the most common first-time error.
```

## 练习

1. 说明：**Easy.** Implement `softmax` masking so padding 词元 in the encoder get 注意力 weight zero. Test on a batch 使用 variable-length sequences.
2. 说明：**Medium.** Add multi-head 注意力 to the Luong `general` form. Split `d_h` 到 `n_heads` groups, run 注意力 per head, concatenate. Verify the single-head case matches your earlier implementation.
3. **Hard.** Train a GRU encoder-decoder 使用 Bahdanau 注意力 on the toy copy 任务 从 lesson 09. Plot 准确率 vs 序列 length. Compare against the no-注意力 基线. You should see the gap widen as length grows, confirming 注意力 lifts the bottleneck.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|注意力|Looking at things|Weighted average of a 值 序列, weights computed 从 a 查询-key 相似度.|
|Query, Key, 值|QKV|说明：Three projections: Q asks, K is what to match, V is what to return.|
|Additive 注意力|Bahdanau|Feed-forward score: `v^T tanh(W q + U k)`.|
|Multiplicative 注意力|Luong dot / general|Score is `q^T k` 或 `q^T W k`. Cheaper, same 准确率 on most tasks.|
|Alignment 矩阵|这个 pretty picture|说明：注意力 weights as a `(T_dec, T_enc)` grid. Read it to see what the 模型 attended to.|

## 延伸阅读

- 说明：[说明：Bahdanau, Cho, Bengio (2014). Neural Machine Translation by Jointly Learning to Align 与 Translate](https://arxiv.org/abs/1409.0473)，the paper.
- 说明：[说明：Luong, Pham, Manning (2015). Effective Approaches to 注意力-based Neural Machine Translation](https://arxiv.org/abs/1508.04025)，the three score variants 与 their comparison.
- 说明：[Jain 与 Wallace (2019). 注意力 is not Explanation](https://arxiv.org/abs/1902.10186)，the interpretability caveat.
- 说明：[Dive 到 Deep Learning，Bahdanau 注意力](https://d2l.ai/chapter_attention-mechanisms-and-transformers/bahdanau-attention.html)，runnable walkthrough 使用 PyTorch.
