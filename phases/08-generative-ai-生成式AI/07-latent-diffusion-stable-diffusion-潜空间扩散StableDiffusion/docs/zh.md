# 潜伏 Diffusion 和 Stable Diffusion

> 512×512 图像上的像素空间扩散是一种计算战争犯罪。罗姆巴赫等人。 (2022) 注意到您不需要所有 786k 维度来生成图像 - 您需要足够的维度来捕获语义结构，并为其余部分提供一个单独的解码器。在 VAE 的潜在空间内运行扩散。这个想法就是 Stable Diffusion。

**类型：** 建造
**语言：** Python
**先修：** 8·02期（VAE）、8·06期（DDPM）、7·09期（ViT）
**时间：** 〜75分钟

## 问题

512² 的像素空间扩散意味着 U-Net 在形状为 `[B, 3, 512, 512]` 的张量上运行。对于 500M 参数的 U-Net，每个采样步骤约为 100 GFLOPS。五十步相当于每个图像 5 TFLOPS。对十亿张图像进行训练，计算费用是荒谬的。

这些 FLOP 中的大多数都用于通过网络推送感知上不重要的细节 - 有损 VAE 可以压缩掉的高频纹理。 Rombach 的想法：训练 VAE 一次（*第一阶段*），将其冻结，并完全在 4 通道 64×64 潜在空间中运行扩散（*第二阶段*）。同样的U网。 1/16th 像素。在同等质量下，失败次数减少约 64 倍。

这是Stable Diffusion的食谱。 SD 1.x / 2.x 在 `64×64×4` 潜在网络上使用 860M U-Net，SDXL 在 `128×128×4` 上使用 2.6B U-Net，SD3 将 U-Net 替换为具有流量匹配的 Diffusion Transformer (DiT)。 Flux.1-dev（黑森林实验室，2024）发布了 12B 参数 DiT-MMDiT。所有这些都在相同的两级基板上运行。

## 概念

![潜在扩散：VAE压缩+潜在空间扩散](../assets/latent-diffusion.svg)

**两个阶段，分别训练。**

1. **第 1 阶段 — VAE。** 编码器 `E(x) → z`，解码器 `D(z) → x`。目标压缩：每个空间轴 8 倍下采样 + 调整通道，使总潜在大小约为像素数的 1/16th。损失 = 重建（L1 + LPIPS 感知）+ KL（权重较小，因此 `z` 不会被迫太高斯，因为我们不需要从 `z` 进行精确采样）。通常使用对抗性损失进行训练，因此解码的图像非常清晰。

2. **第 2 阶段 — `z` 上的扩散。** 将 `z = E(x_real)` 视为数据。训练 U-Net（或 DiT）对 `z_t` 进行去噪。推理时：通过扩散采样 `z_0`，然后是 `x = D(z_0)`。

**文本调节。** 两个附加组件。冻结文本编码器（用于 SD 1.x 的 CLIP-L、用于 SD 2/XL 的 CLIP-L+OpenCLIP-G、用于 SD3 和 Flux 的 T5-XXL）。交叉注意力注入：每个 U-Net 块都采用 `[Q = image features, K = V = text tokens]` 并将它们混合在一起。词元是文本影响图像的唯一方式。

**损失函数与第 06 课相同。** 相同的 DDPM / 噪声匹配 MSE 的流量。您只需交换数据域即可。

## 架构变体

| 模型 | 年 | 骨干 | 潜形 | 文本编码器 | 参数 |
|-------|------|----------|--------------|--------------|--------|
| 标准差1.5 | 2022年 | 优网 | 64×64×4 | CLIP-L（77 个词元） | 860M |
| 标清2.1 | 2022年 | 优网 | 64×64×4 | OpenCLIP-H | 865M |
| SDXL | 2023年 | U网+精炼机 | 128×128×4 | CLIP-L + OpenCLIP-G | 2.6B+6.6B |
| SDXL-涡轮 | 2023年 | 蒸馏的 | 128×128×4 | 相同的 | 1-4步采样 |
| SD3 | 2024年 | MMDiT（多模态 DiT） | 128×128×16 | T5-XXL + CLIP-L + CLIP-G | 2B/8B |
| Flux.1-dev | 2024年 | MMDiT | 128×128×16 | T5-XXL + 夹子-L | 12B |
| Flux.1-施奈尔 | 2024年 | MMDiT 蒸馏 | 128×128×16 | T5-XXL + 夹子-L | 12B，1-4步 |

趋势：用 DiT（潜在补丁上的变压器）取代 U-Net，缩放文本编码器（T5 击败 CLIP 以实现快速粘附），增加潜在通道（4 → 16 提供更多细节空间）。

```figure
noise-schedule
```

## 构建它

`code/main.py` 在第 06 课的 DDPM 顶部堆叠了一个玩具 1-D“VAE”（身份编码器 + 解码器，用于演示；真正的 VAE 将是一个转换网络），并添加了具有无分类器指导的类调节。它表明，无论您运行的是原始一维值还是编码值，相同的扩散损失都有效，这是关键的见解。

### 第1步：encoder/decoder

```python
def encode(x):    return x * 0.5          # toy "compression" to smaller scale
def decode(z):    return z * 2.0
```

真正的 VAE 已经训练有素。对于教育学来说，这个线性图足以表明扩散在 `z` 上进行，而不关心原始数据空间。

### 步骤 2：`z` 空间中的扩散

和第06课一样的DDPM。网上看到的数据是`z = E(x)`。采样`z_0`后，用`D(z_0)`解码。

### 步骤 3：无分类器指导

在训练期间，有 10% 的时间删除类标签（用空标记替换）。推理时，计算 `ε_cond` 和 `ε_uncond`，然后：

```python
eps_cfg = (1 + w) * eps_cond - w * eps_uncond
```

`w = 0` = 无引导（完全多样性），`w = 3` = 默认，`w = 7+` = 饱和/过度锐利。

### 第 4 步：文本调节（概念，而非代码）

用冻结的文本编码器输出替换类标签。通过交叉注意力将文本嵌入输入到 U-Net：

```python
h = h + CrossAttention(Q=h, K=text_embed, V=text_embed)
```

这是类条件扩散模型和 Stable Diffusion 之间唯一的实质性区别。

## 常见陷阱

- **VAE 比例不匹配。** SD 1.x VAE 在编码后应用了缩放常数 (`scaling_factor ≈ 0.18215`)。忘记这一点会使 U-Net 训练的潜在变量的方差非常错误。每个检查站都会运送一个。
- **文本编码器默默地错误。** SD3 需要具有 >=128 个标记的 T5-XXL，并且回退到仅 CLIP 是有损的。始终检查 `use_t5=True` 或提示保真陨石坑。
- **混合潜在空间。** SDXL、SD3、Flux 都使用不同的 VAE。在 SDXL 潜伏上训练的 LoRA 不适用于 SD3。 Hugging Face 扩散器 0.30+ 拒绝加载不匹配的检查点。
- **CFG 太高。** `w > 10` 会产生饱和、油腻的图像，并以多样性为代价过拟合提示。最佳位置是 `w = 3-7`。
- **负面提示泄漏。**空的负面提示成为空词元；填充的否定提示变为 `ε_uncond`。这些并不相同；一些管道默默地默认为空。

## 使用它

2026 年生产堆栈：

| 目标 | 推荐骨干 |
|--------|----------------------|
| 窄域、配对数据、从头开始训练模型 | SDXL 微调（LoRA / 完整）— 最快发货 |
| 开放域文本到图像、开放权重 | Flux.1-dev（12B，Apache/非商业）或SD3.5-Large |
| 最快的推理，开放权重 | Flux.1-schnell（1-4 步骤，Apache）或 SDXL-Lightning |
| 最佳及时依从性，托管 | GPT-图像 / DALL-E 3（静止）、Midjourney v7、Imagen 4 |
| 编辑工作流程 | Flux.1-Kontext（2024 年 12 月）— 本身接受图像 + 文本 |
| 研究，基线 | SD 1.5，古老但经过充分研究 |

## 交付它

保存`outputs/skill-sd-prompter.md`。 Skill 采用文本提示 + 目标样式和输出：模型 + 检查点、CFG 比例、采样器、否定提示、分辨率、可选的 ControlNet/IP-Adapter 组合以及每步 QA 检查表。

## 练习

1. **简单。** 在 `w ∈ {0, 1, 3, 7, 15}` 指导下运行 `code/main.py`。按类别记录平均样本。类别均值在 `w` 时偏离真实数据均值？
2. **中。** 将玩具线性编码器替换为具有重构损失的 tanh-MLP encoder/decoder 对。重新训练新潜伏的扩散。样品质量是否发生变化？
3. **困难。** 使用扩散器设置真实的 Stable Diffusion 推理：加载 `sdxl-base`，在 CFG=7 的情况下运行 30 个欧拉步骤，计时。现在切换到 `sdxl-turbo`，步骤 4，CFG=0。相同的主题，不同的质量，描述发生了什么变化以及原因。

## 关键术语

| 学期 | 人们怎么说 | 它实际上意味着什么 |
|------|-----------------|-----------------------|
| 第一阶段 | “VAE” | 训练encoder/decoder对；将 512² 压缩为 64²。 |
| 第二阶段 | “U网” | 潜在空间上方的 Diffusion 模型。 |
| CFG | 《指导量表》 | `(1+w)·ε_cond - w·ε_uncond`；调整调节强度。 |
| 空词元 | “空提示嵌入” | 用于 `ε_uncond` 的无条件嵌入。 |
| 交叉注意力 | “文本是如何进入的” | 每个 U-Net 块都处理文本标记 K 和 V。 |
| 迪特 | “Diffusion Transformer” | 用潜在补丁上的变压器替换 U-Net；规模更好。 |
| MMDiT | “多模态 DiT” | SD3的架构：共同关注的文本和图像流。 |
| VAE 比例因子 | 《神奇数字》 | 将潜在变量除以约 5.4，因此扩散在单位方差空间中进行。 |

## 生产说明：在 8GB 消费级 GPU 上运行 Flux-12B

参考 Flux 集成是规范的“我有一个消费级 GPU，我可以发货吗？”食谱。诀窍是将相同的三旋钮配方生产推理文献列表应用于扩散 DiT：

1. **交错加载。** Flux 具有三个不需要在 VRAM 中共存的网络：T5-XXL 文本编码器（fp32 中约 10 GB）、CLIP-L（小）、12B MMDiT 和 VAE。首先对提示进行编码，*删除*编码器，加载 DiT，去噪，*删除* DiT，加载 VAE，解码。消费类 8GB GPU 一次仅适合一个阶段。
2. **通过bitsandbytes进行4位量化。** T5编码器和DiT上的`BitsAndBytesConfig(load_in_4bit=True, bnb_4bit_compute_dtype=torch.bfloat16)`。内存减少 8 倍，根据 Aritra 的基准（在笔记本中链接），文本到图像的质量下降是难以察觉的。
3. **CPU 卸载。** 随着每次前向传递的进行，`pipe.enable_model_cpu_offload()` 在 CPU 和 GPU 之间自动交换模块。增加 10-20% 的延迟，但会使管道继续运行。

内存计算为：`10 GB T5 / 8 = 1.25 GB` 量化、`12 B params × 0.5 bytes = ~6 GB` 量化 DiT，加上激活。用 stas00 的话来说，这是 TP=1 推理的极端，无模型并行性，最大量化。对于生产环境，您可以在 H100 上运行 TP=2 或 TP=4；对于一台开发笔记本电脑，这就是秘诀。

## 延伸阅读

- [罗姆巴赫等人。 （2022）。使用潜在 Diffusion 模型进行高分辨率图像合成](https://arxiv.org/abs/2112.10752) — Stable Diffusion。
- [波德尔等人。 （2023）。 SDXL：改进用于高分辨率图像合成的潜在 Diffusion 模型](https://arxiv.org/abs/2307.01952) — SDXL。
- [皮布尔斯和谢 (2023)。可扩展 Diffusion 型号，带 Transformers (DiT)](https://arxiv.org/abs/2212.09748) — DiT。
- [埃塞尔等人。 （2024）。用于高分辨率图像合成的缩放整流流 Transformers](https://arxiv.org/abs/2403.03206) — SD3、MMDiT。
- [Ho 和 Salimans (2022)。无分类器 Diffusion 指导](https://arxiv.org/abs/2207.12598) — CFG。
- [实验室（2024）。 Flux.1 — 黑森林实验室公告](https://blackforestlabs.ai/announcing-black-forest-labs/) — Flux.1 系列。
- [Hugging Face Diffusers 文档](https://huggingface.co/docs/diffusers/index) — 上述每个检查点的参考实现。
