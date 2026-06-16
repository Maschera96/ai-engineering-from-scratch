# 从零构建 Transformer 综合实战

> 把注意力、位置编码、残差、前馈、归一化、采样和训练循环接成一个可运行的小模型。

**Type:** Build
**Languages:** Python
**Prerequisites:** Phase 7 · 01 through 13. Don't skip.
**Time:** ~120 minutes

## The Problem

本节说明原始问题和为什么需要当前架构。核心动机来自英文源文档, 但这里用简体中文重新表述: 传统做法在并行性、长距离依赖、内存或任务适配上有明显瓶颈；本课的 Transformer 组件通过注意力、位置编码、残差、前馈、缓存或稀疏化等机制解决这些瓶颈。

## The Concept

![Transformer-from-scratch block diagram](../assets/capstone.svg)

概念上，把输入先表示成词元序列，再让模型在序列位置之间交换信息。注意力负责跨位置混合，前馈网络负责逐位置变换，残差连接保留信息流，归一化保持训练稳定。不同 lesson 只是在这个骨架上改变一个关键部件。

```
input tokens (B, N)
   │
   ▼
token embedding + positional embedding  ◀── Lesson 04 (RoPE option)
   │
   ▼
┌──── block × L ────────────────────┐
│  RMSNorm                          │  ◀── Lesson 05
│  MultiHeadAttention (causal)      │  ◀── Lesson 03 + 07 (causal mask)
│  residual                         │
│  RMSNorm                          │
│  SwiGLU FFN                       │  ◀── Lesson 05
│  residual                         │
└────────────────────────────────── ┘
   │
   ▼
final RMSNorm
   │
   ▼
lm_head (tied to token embedding)
   │
   ▼
logits (B, N, V)
   │
   ▼
shift-by-one cross-entropy            ◀── Lesson 07
```


## Build It

跟随 `code/main.py`。实现保持小而透明，优先展示机制而不是追求生产性能。保留英文源中的代码片段和命令，确保读者能按同样路径运行。

### Step 1: data

这一小步把概念落到可执行代码中。重点检查张量形状、掩码方向、缓存增长、路由选择或采样输出是否符合预期。

### Step 2: model

这一小步把概念落到可执行代码中。重点检查张量形状、掩码方向、缓存增长、路由选择或采样输出是否符合预期。

### Step 3: training loop

这一小步把概念落到可执行代码中。重点检查张量形状、掩码方向、缓存增长、路由选择或采样输出是否符合预期。

### Step 4: sample

这一小步把概念落到可执行代码中。重点检查张量形状、掩码方向、缓存增长、路由选择或采样输出是否符合预期。

### Step 5: read the output

这一小步把概念落到可执行代码中。重点检查张量形状、掩码方向、缓存增长、路由选择或采样输出是否符合预期。

```python
text = open("tinyshakespeare.txt").read()
chars = sorted(set(text))
stoi = {c: i for i, c in enumerate(chars)}
itos = {i: c for c, i in stoi.items()}
encode = lambda s: [stoi[c] for c in s]
decode = lambda xs: "".join(itos[x] for x in xs)
```

```python
for step in range(max_steps):
    x, y = get_batch("train")
    logits = model(x)
    loss = F.cross_entropy(logits.view(-1, vocab_size), y.view(-1))
    loss.backward()
    torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
    opt.step()
    opt.zero_grad()
```

```
ROMEO:
Away and mild will not thy friend, that thou shalt wit:
The chief that well shame and hath been his friends,
...
```


## Use It

在生产代码中通常直接使用 PyTorch、HuggingFace、vLLM、Flash Attention 或对应模型库。保留 API 名称、模型名、命令和文件路径不翻译，方便复制运行。


## Ship It

查看 `outputs/skill-transformer-review.md`。这个产物把本课方法转成可复用的 skill 或 prompt，用于真实项目中的架构选择、配置检查、推理优化或调参。

## Exercises

1. **Easy.** 运行本课代码，确认输出形状、掩码、路由或缓存行为与预期一致。
2. **Medium.** 修改一个关键超参数或组件，比较结果变化，并解释原因。
3. **Hard.** 把本课机制接入一个小型真实任务，测量质量、速度或内存取舍。

## Key Terms

| Term | What people say | What it actually means |
|------|-----------------|-----------------------|
| nanoGPT | "Karpathy's tutorial repo" | Minimal decoder-only transformer training code, ~300 LOC; the canonical reference. |
| tinyshakespeare | "The standard toy corpus" | ~1.1 MB of text; every character-LM tutorial since 2015 uses it. |
| Tied embeddings | "Share input/output matrix" | LM head weight = transpose of token embedding matrix; saves parameters, improves quality. |
| bf16 autocast | "Training precision trick" | Run forward/back in bf16, keep optimizer state in fp32; standard since 2021. |
| Gradient clipping | "Stops spikes" | Cap global grad norm at 1.0; prevents training blowups. |
| Cosine LR schedule | "The 2020+ default" | LR ramps up linearly (warmup) then decays cosine-shaped to 10% of peak. |
| MFU | "Model FLOP Utilization" | Achieved FLOPs / theoretical peak; 40% dense, 30% MoE is strong in 2026. |
| Val loss | "Held-out loss" | Cross-entropy on data the model never saw; overfit detector. |

## Further Reading

- [The Annotated Transformer (Harvard NLP)](https://nlp.seas.harvard.edu/annotated-transformer/) — the classic annotated implementation.
