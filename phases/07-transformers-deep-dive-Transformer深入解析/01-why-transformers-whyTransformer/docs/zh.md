# 为什么选择 Transformer, RNN 的问题

> RNN 一次处理一个词元。Transformer 一次处理所有词元。这个架构选择改变了 2017 年之后深度学习的每条扩展曲线。

**Type:** Learn
**Languages:** Python
**Prerequisites:** Phase 3 (Deep Learning Core), Phase 5 · 09 (Sequence-to-Sequence), Phase 5 · 10 (Attention Mechanism)
**Time:** ~45 minutes

## The Problem

2017 年以前，几乎所有顶尖序列模型都建立在循环神经网络之上。语言、翻译、语音都是如此。LSTM 和 GRU 连续多年统治翻译基准，因为当时大家只有这套工具。

它们有三个致命弱点。第一是串行计算。词元 `t+1` 需要词元 `t` 的隐藏状态，所以时间轴不能并行。一个 1,024 词元序列意味着 1,024 个串行步骤，而 GPU 每个周期可以做大量并行浮点运算。训练时间会随序列长度线性增长，正好违背并行硬件的优势。

第二是梯度消失。50 个词元之前的信息已经穿过 50 次非线性变换。门控循环单元缓解了压缩，但没有消除它。像 “the book I read last summer on a plane to Kyoto was...” 这样的长距离依赖经常失败。

第三是固定宽度隐藏状态。编码器必须先把整个源序列压进一个向量，解码器才能看到它。源序列是 5 个词元还是 500 个词元，瓶颈形状都一样。

2017 年论文 “Attention Is All You Need” 提出了激进做法, 完全去掉循环。让每个位置并行关注其他所有位置。用一次大的矩阵乘法训练，而不是 1,024 次串行计算。

到 2026 年，结果已经覆盖所有模态。语言模型、视觉模型、音频模型、生物模型、机器人模型都在使用同一种块，只是输入不同。

## The Concept

![RNN sequential compute vs Transformer parallel attention](../assets/rnn-vs-transformer.svg)

**循环是瓶颈。** RNN 计算 `h_t = f(h_{t-1}, x_t)`。每一步依赖前一步。你不能在 `h_4` 之前计算 `h_5`。在有上万并行核心的现代 GPU 上，长序列会浪费绝大多数硅片能力。

**注意力像广播。** 自注意力为每一对 `(i, j)` 同时计算 `output_i = sum_j(a_ij * v_j)`。整个 N×N 注意力矩阵在一个批量矩阵乘法中填满。没有步骤依赖另一个步骤。GPU 很适合这种工作。

**加速不是常数。** 它是 `O(N)` 串行深度和 `O(1)` 串行深度的差别。实践里，在 N=512 的同等硬件上，Transformer 每个 epoch 通常快 5 到 10 倍。序列越长差距越大，直到碰到注意力 `O(N²)` 内存墙，Flash Attention 后来缓解了这个问题，见 Lesson 12。

**Transformer 的代价。** 注意力内存按 `O(N²)` 增长。2K 上下文没问题，128K 上下文就需要滑动窗口、RoPE 外推、Flash Attention 分块或线性注意力变体。循环在时间和内存上都是 `O(N)`。Transformer 用内存换时间，再靠并行把时间赢回来。

**归纳偏置改变。** RNN 假设局部性和近因性。Transformer 不做这种假设，每一对位置都可能形成注意力。这就是 Transformer 需要更多数据才能训好，但一旦有数据就能扩展得更远的原因。Chinchilla 在 2022 年形式化了这一点, 给定足够词元，同参数量 Transformer 总能超过 RNN。

## Build It

这里不写神经网络。我们用数值模拟核心瓶颈，让你在笔记本上亲自感受差距。

### Step 1: measure serial depth

看 `code/main.py`。我们写两个函数。一个把序列编码成加法链，像 RNN 一样串行。另一个把它编码成并行归约，像注意力一样广播。数学相同，依赖图不同。

```python
def rnn_style(xs):
    h = 0.0
    for x in xs:
        h = 0.9 * h + x   # can't parallelize: h depends on previous h
    return h

def attention_style(xs):
    return sum(xs) / len(xs)  # every x is independent
```

我们测量最长 100,000 元素序列上的时间。RNN 版本是 O(N)，并且走单条 CPU 流水线。即使在纯 Python 中，长度 ≥ 1,000 时，注意力风格的归约也会更快，因为 Python 的 `sum()` 在 C 中实现，不需要每步解释器开销。

### Step 2: count theoretical operations

两个算法都做 N 次加法。差别是依赖深度, 下一步开始前必须按顺序完成多少操作。RNN 深度是 N。注意力可以用树形归约达到 log(N)，或者用并行扫描达到 1。决定 GPU 时间的是深度，不是操作数。

### Step 3: empirical scaling on long sequences

程序会打印计时表，让 O(N) 差距变得可见。在 2026 年的 Mac 笔记本上，1,000 以下的序列太快而难以测量。100,000 长度会显示清晰线性扫描。把它扩展到 16,384 词元、12 层 LSTM 等价模型，就能看出为什么 2016 年的训练墙钟时间是阻碍。

## Use It

2026 年仍然选择 RNN 的场景:

| Situation | Pick |
|-----------|------|
| Streaming inference, one token at a time, constant memory | RNN or state-space model (Mamba, RWKV) |
| Very long sequences (>1M tokens) where attention memory explodes | Linear attention, Mamba 2, Hyena |
| Edge device with no matmul accelerator | Depthwise-separable RNN still wins on FLOPs/watt |
| Anything else (training, batched inference, context up to 128K) | Transformer |

Mamba 这类状态空间模型本质上是带结构化参数的 RNN，获得了两边的优点, `O(N)` 扫描内存，以及通过选择性扫描实现的并行训练。它们用更好的长上下文扩展拿回了 Transformer 约 90% 的质量。2026 年多数前沿实验室训练混合 SSM+Transformer 模型，例如 Jamba 和 Samba。循环没有死，它变成了组件。

## Ship It

看 `outputs/skill-architecture-picker.md`。这个 skill 会根据长度、吞吐和训练预算约束，为新的序列问题选择架构。对于超过 1B 词元的训练，如果推荐纯 RNN，它必须说明取舍，否则应拒绝这样推荐。

## Exercises

1. **Easy.** 从 `code/main.py` 取出 `rnn_style`，把标量隐藏状态换成长为 64 的隐藏状态向量。重新测量。串行开销会如何随隐藏维度增长？
2. **Medium.** 用纯 Python 实现并行前缀和，也就是 Hillis-Steele scan。验证它在长度 1024 上和串行扫描得到相同数值。统计深度。
3. **Hard.** 把注意力风格归约移植到 GPU 上的 PyTorch。序列长度从 64 扫到 65,536，测量两种方法的时间。画图并解释曲线形状。

## Key Terms

| Term | What people say | What it actually means |
|------|-----------------|-----------------------|
| Recurrence | “RNN 是串行的” | 步骤 `t` 依赖步骤 `t-1` 的计算，迫使时间轴串行执行。 |
| Serial depth | “计算图有多深” | 最长依赖操作链，即使硬件无限也限制墙钟时间。 |
| Attention | “让词元互相看” | 加权和 `sum_j a_ij v_j`，其中 `a_ij` 来自位置 i 和 j 的相似度分数。 |
| Context window | “模型能看到多少” | 注意力层可接收的位置数量，二次内存成本在这里增长。 |
| Inductive bias | “架构里内置的假设” | 对数据形态的先验。CNN 假设平移不变性，RNN 假设近因性。 |
| State-space model | “背后有代数的 RNN” | 通过结构化状态空间矩阵参数化的循环，可并行训练。 |
| Quadratic bottleneck | “为什么上下文这么贵” | 注意力内存随序列长度为 `O(N²)`。Flash Attention 降低常数，不改变阶数。 |

## Further Reading

- [Vaswani et al. (2017). Attention Is All You Need](https://arxiv.org/abs/1706.03762), 主流 NLP 中终结循环的论文。
- [Bahdanau, Cho, Bengio (2014). Neural MT by Jointly Learning to Align and Translate](https://arxiv.org/abs/1409.0473), 注意力诞生于这里，当时仍接在 RNN 上。
- [Hochreiter, Schmidhuber (1997). Long Short-Term Memory](https://www.bioinf.jku.at/publications/older/2604.pdf), 原始 LSTM 论文。
- [Gu, Dao (2023). Mamba: Linear-Time Sequence Modeling with Selective State Spaces](https://arxiv.org/abs/2312.00752), 对 Transformer 的现代循环式回答。
