# OCR 和文档理解

> OCR 是一个三阶段管道 - 检测文本框、识别字符，然后将其布局。每个现代 OCR 系统都会对这些阶段进行重新排序或合并。

**类型：** Learn + Use
**语言：** Python
**先修：** Phase 4 Lesson 06 (Detection), Phase 7 Lesson 02 (Self-Attention)
**时间：** 约 45 分钟

## 学习目标

- 追踪经典 OCR 管道（检测 -> 识别 -> 布局）和现代端到端替代方案（Donut、Qwen-VL-OCR）
- 为序列到序列 OCR 训练实施 CTC（连接主义时间分类）损失
- 使用PaddleOCR或EasyOCR进行生产文档解析，无需培训
- 区分 OCR、布局解析和文档理解 - 并为每个任务选择正确的工具

## 问题

充满文字的图像无处不在：收据、发票、身份证、扫描书籍、表格、白板、标牌、屏幕截图。从它们中提取结构化数据——不仅仅是字符，而是“这是总量”——是价值最高的应用视觉问题之一。

该领域分为三个技能层：

1. **OCR 正确**：将像素转换为文本。
2. **布局解析**：将 OCR 输出分组为区域（标题、正文、表格、标题）。
3. **文档理解**：从布局中提取结构化字段（“invoice_total = $42.50”）。

每一层都有经典和现代的方法，“我想要图像中的文本”和“我需要这张收据的总金额”之间的差距比大多数团队意识到的要大。

## 概念

### 经典管道

```mermaid
flowchart LR
    IMG["图像"] --> DET["文本检测<br/>（DB、EAST、CRAFT）"]
    DET --> BOX["Word/line<br/>边界框"]
    BOX --> CROP["裁剪每个区域"]
    CROP --> REC["认可<br/>（CRNN + CTC）"]
    REC --> TXT["文本字符串"]
    TXT --> LAY["布局<br/>排序"]
    LAY --> OUT["阅读顺序文本"]

    style DET fill:#dbeafe,stroke:#2563eb
    style REC fill:#fef3c7,stroke:#d97706
    style OUT fill:#dcfce7,stroke:#16a34a
```

- **文本检测** 生成每行或每单词的四边形。
- **识别**将每个区域裁剪到固定高度，运行 CNN + BiLSTM + CTC 来生成字符序列。
- **布局** 重建阅读顺序（拉丁语从上到下、从左到右；阿拉伯语、日语不同）。

### 一段话中的 CTC

OCR 识别从固定长度的特征图生成可变长度的序列。 CTC（Graves et al., 2006）让您可以在不进行字符级对齐的情况下进行训练。模型在每个时间步输出（词汇+空白）上的分布； CTC 损失边缘化了所有在合并重复和删除空白后减少到目标文本的对齐。

```
raw output: "h h h _ _ e e l l _ l l o _ _"
after merge repeats and remove blanks: "hello"
```

CTC 是 CRNN 在 2015 年发挥作用的原因，并且在 2026 年仍然训练大多数生产 OCR 模型。

### 现代端到端模型

- **Donut** (Kim et al., 2022) — ViT 编码器 + 文本解码器；读取图像并直接发出 JSON。没有文本检测器，没有布局模块。
- **TrOCR** — ViT + 用于行级 OCR 的Transformer解码器。
- **Qwen-VL-OCR / InternVL** — 针对 OCR 任务进行微调的完整视觉语言模型； 2026 年复杂文档的最佳准确率。
- **PaddleOCR** — 成熟生产包中的经典 DB + CRNN 管道；仍然是开源的主力。

端到端模型需要更多的数据和计算，但跳过多级管道的误差累积。

### 布局解析

对于结构化文档，运行布局检测器（LayoutLMv3、DocLayNet）来标记每个区域：标题、段落、图形、表格、脚注。然后，读取顺序变为“按布局顺序迭代区域，连接”。

对于表单，请使用 **键值提取** 模型（Donut 用于视觉效果丰富的文档，LayoutLMv3 用于普通扫描）。他们采用图像+检测到的文本+位置并预测结构化键值对。

### 评估指标

- **字符错误率 (CER)** — 编辑距离/参考长度。越低越好。生产目标：干净扫描时 < 2%。
- **字错误率 (WER)** — 在字级别相同。
- **结构化字段上的 F1** — 用于键值任务；测量`{invoice_total: 42.50}`是否正确显示。
- **JSON 上的编辑距离** — 用于端到端文档解析； Donut 论文引入了标准化树编辑距离。

## Build It

### 步骤1：CTC损失+贪婪解码器

```python
import torch
import torch.nn as nn
import torch.nn.functional as F


def ctc_loss(log_probs, targets, input_lengths, target_lengths, blank=0):
    """
    log_probs:      (T, N, C) log-softmax over vocab including blank at index 0
    targets:        (N, S) int targets (no blanks)
    input_lengths:  (N,) per-sample time steps used
    target_lengths: (N,) per-sample target length
    """
    return F.ctc_loss(log_probs, targets, input_lengths, target_lengths,
                      blank=blank, reduction="mean", zero_infinity=True)


def greedy_ctc_decode(log_probs, blank=0):
    """
    log_probs: (T, N, C) log-softmax
    returns: list of index sequences (blanks removed, repeats merged)
    """
    preds = log_probs.argmax(dim=-1).transpose(0, 1).cpu().tolist()
    out = []
    for seq in preds:
        decoded = []
        prev = None
        for idx in seq:
            if idx != prev and idx != blank:
                decoded.append(idx)
            prev = idx
        out.append(decoded)
    return out
```

`F.ctc_loss` 使用高效的 CuDNN 实现（如果可用）。贪婪解码器比波束搜索更简单，并且通常在 1% CER 之内。

### 第 2 步：微型 CRNN 识别器

用于行 OCR 的最小 CNN + BiLSTM。

```python
class TinyCRNN(nn.Module):
    def __init__(self, vocab_size=40, hidden=128, feat=32):
        super().__init__()
        self.cnn = nn.Sequential(
            nn.Conv2d(1, feat, 3, 1, 1), nn.BatchNorm2d(feat), nn.ReLU(inplace=True),
            nn.MaxPool2d(2),
            nn.Conv2d(feat, feat * 2, 3, 1, 1), nn.BatchNorm2d(feat * 2), nn.ReLU(inplace=True),
            nn.MaxPool2d(2),
            nn.Conv2d(feat * 2, feat * 4, 3, 1, 1), nn.BatchNorm2d(feat * 4), nn.ReLU(inplace=True),
            nn.MaxPool2d((2, 1)),
            nn.Conv2d(feat * 4, feat * 4, 3, 1, 1), nn.BatchNorm2d(feat * 4), nn.ReLU(inplace=True),
            nn.MaxPool2d((2, 1)),
        )
        self.rnn = nn.LSTM(feat * 4, hidden, bidirectional=True, batch_first=True)
        self.head = nn.Linear(hidden * 2, vocab_size)

    def forward(self, x):
        # x: (N, 1, H, W)
        f = self.cnn(x)                # (N, C, H', W')
        f = f.mean(dim=2).transpose(1, 2)  # (N, W', C)
        h, _ = self.rnn(f)
        return F.log_softmax(self.head(h).transpose(0, 1), dim=-1)  # (W', N, vocab)
```

固定高度输入（CNN max-pools 高度为 1）。宽度是CTC的时间维度。

### 第 3 步：合成 OCR

Generate black-on-white digit strings for an end-to-end smoke test.

```python
import numpy as np

def synthetic_line(text, height=32, char_width=16):
    W = char_width * len(text)
    img = np.ones((height, W), dtype=np.float32)
    for i, c in enumerate(text):
        x = i * char_width
        shade = 0.0 if c.isalnum() else 0.5
        img[6:height - 6, x + 2:x + char_width - 2] = shade
    return img


def build_batch(strings, vocab):
    H = 32
    W = 16 * max(len(s) for s in strings)
    imgs = np.ones((len(strings), 1, H, W), dtype=np.float32)
    target_lengths = []
    targets = []
    for i, s in enumerate(strings):
        imgs[i, 0, :, :16 * len(s)] = synthetic_line(s)
        ids = [vocab.index(c) for c in s]
        targets.extend(ids)
        target_lengths.append(len(ids))
    return torch.from_numpy(imgs), torch.tensor(targets), torch.tensor(target_lengths)


vocab = ["_"] + list("0123456789abcdefghijklmnopqrstuvwxyz")
imgs, targets, lengths = build_batch(["hello", "world"], vocab)
print(f"images: {imgs.shape}   targets: {targets.shape}   lengths: {lengths.tolist()}")
```

真实的 OCR 数据集添加字体、噪声、旋转、模糊和颜色。上面的管道是相同的。

### 第四步：训练草图

```python
model = TinyCRNN(vocab_size=len(vocab))
opt = torch.optim.Adam(model.parameters(), lr=1e-3)

for step in range(200):
    strings = ["abc" + str(step % 10)] * 4 + ["xyz" + str((step + 1) % 10)] * 4
    imgs, targets, target_lens = build_batch(strings, vocab)
    log_probs = model(imgs)  # (W', 8, vocab)
    input_lens = torch.full((8,), log_probs.size(0), dtype=torch.long)
    loss = ctc_loss(log_probs, targets, input_lens, target_lens, blank=0)
    opt.zero_grad(); loss.backward(); opt.step()
```

Loss should drop from 约 3 to 约 0.2 over 200 steps on this trivial synthetic data.

## Use It

Three production paths:

- **PaddleOCR** — 成熟、快速、多语言。一行用法：`paddleocr.PaddleOCR(lang="en").ocr(image_path)`。
- **EasyOCR** — Python-native, multilingual, PyTorch backbone.
- **Tesseract** — 经典；当模型遇到困难时，对于旧的扫描文档仍然有用。

对于端到端文档解析，请使用 Donut 或 VLM：

```python
from transformers import DonutProcessor, VisionEncoderDecoderModel

processor = DonutProcessor.from_pretrained("naver-clova-ix/donut-base-finetuned-cord-v2")
model = VisionEncoderDecoderModel.from_pretrained("naver-clova-ix/donut-base-finetuned-cord-v2")
```

对于具有可重复结构的收据、发票和表格，请微调 Donut。对于任意文档或具有推理特征的 OCR，当前默认使用像 Qwen-VL-OCR 这样的 VLM。

## Ship It

本课产生：

- `outputs/prompt-ocr-stack-picker.md` — 根据文档类型、语言和结构选择 Tesseract / PaddleOCR / Donut / VLM-OCR 的提示。
- `outputs/skill-ctc-decoder.md` — a skill that writes greedy and beam-search CTC decoders from scratch, including length normalisation.

## 练习

1. **（简单）** 使用 5 位随机数字字符串训练 TinyCRNN 500 步。报告保留集的 CER。
2. **（中）** 用波束搜索替换贪婪解码（beam_width=5）。报告 CER 增量。集束搜索在哪些输入上获胜？
3. **（困难）** 在一组 20 张收据上使用 PaddleOCR，提取行项目，并针对 {item_name,price} 对的手动标记的基本事实计算 F1。

## 关键术语

| 学期 | 人们怎么说 | 它实际上意味着什么 |
|------|----------------|----------------------|
| 光学字符识别 | “来自像素的文本” | 将图像区域转换为字符序列 |
| CTC | “无对齐损失” | 训练没有每个时间步标签的序列模型的损失；边缘化超过对齐 |
| 循环神经网络 | 《经典OCR模型》 | Conv特征提取器+BiLSTM+CTC；生产中仍使用 2015 年基准 |
| 油炸圈饼 | “端到端OCR” | ViT编码器+文本解码器；直接从图像发出 JSON |
| 布局解析 | “查找区域” | 检测并标记文档中的Title/Table/Figure/Paragraph区域 |
| 阅读顺序 | “文本序列” | 将识别的区域排序成句子；对于拉丁语来说很简单，对于混合布局来说很重要 |
| 核证减排量 /WER | “错误率” | 字符或单词粒度的编辑距离/参考长度 |
| VLM-OCR | “读法学硕士” | 针对 OCR 任务进行训练或提示的视觉语言模型；当前复杂文档的 SOTA |

## 延伸阅读

- [CRNN (Shi et al., 2015)](https://arxiv.org/abs/1507.05717) — 原始CNN+RNN+CTC架构
- [CTC (Graves et al., 2006)](https://www.cs.toronto.edu/约 graves/icml_2006.pdf) — 原始 CTC 论文；算法思想密密麻麻
- [Donut (Kim et al., 2022)](https://arxiv.org/abs/2111.15664) — 无 OCR 文档理解转换器
- [PaddleOCR](https://github.com/PaddlePaddle/PaddleOCR) — 开源生产 OCR 堆栈
