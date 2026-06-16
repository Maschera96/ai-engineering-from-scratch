# GPU 设置和云

> CPU 训练非常适合学习。真实训练需要 GPU。

**类型：** ** Build
**语言：** ** Python
**先修：** ** 第 0 阶段，第 01 课
**时间：** ** 约 45 分钟

## 学习目标

- 使用 `nvidia-smi` 和 PyTorch 的 CUDA API 验证本地 GPU 可用性
- 使用 T4 GPU 配置 Google Colab 以进行基于云的免费实验
- 对 CPU 与 GPU 上的矩阵乘法进行基准测试并测量加速比
- 使用 fp16 经验法则估计适合您的 VRAM 的最大模型

＃＃ 问题

第 1-3 阶段的大多数课程在 CPU 上运行良好。但一旦开始训练 CNN、Transformer 或 LLM（第 4 阶段以上），您就需要 GPU 加速。在 CPU 上运行 8 小时的训练在 GPU 上运行需要 10 分钟。

您有三个选择：本地 GPU、云 GPU 或 Google Colab（免费）。

## 概念

```
Your options:

1. Local NVIDIA GPU
   Cost: $0 (you already have it)
   Setup: Install CUDA + cuDNN
   Best for: Regular use, large datasets

2. Google Colab (free tier)
   Cost: $0
   Setup: None
   Best for: Quick experiments, no GPU at home

3. Cloud GPU (Lambda, RunPod, Vast.ai)
   Cost: $0.20-2.00/hr
   Setup: SSH + install
   Best for: Serious training, large models
```

## Build It

### 选项 1：本地 NVIDIA GPU

检查您是否有：

```bash
nvidia-smi
```

安装带有 CUDA 的 PyTorch：

```python
import torch

print(f"CUDA available: {torch.cuda.is_available()}")
print(f"CUDA version: {torch.version.cuda}")
if torch.cuda.is_available():
    print(f"GPU: {torch.cuda.get_device_name(0)}")
    print(f"Memory: {torch.cuda.get_device_properties(0).total_memory / 1e9:.1f} GB")
```

### 选项 2：Google Colab

1. 前往 [colab.research.google.com](https://colab.research.google.com)
2.运行时 > 更改运行时类型 > T4 GPU
3.运行`!nvidia-smi`进行验证

将本课程的Notebook直接上传到 Colab。

### 选项 3：云 GPU

对于 Lambda Labs、RunPod 或 Vast.ai：

```bash
ssh user@your-gpu-instance

pip install torch torchvision torchaudio
python -c "import torch; print(torch.cuda.get_device_name(0))"
```

### 没有 GPU？没问题。

大多数课程都是针对 CPU 的。需要 GPU 的会这么说并包含 Colab 链接。

```python
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
print(f"Using: {device}")
```

## 构建：GPU 与 CPU 基准测试

```python
import torch
import time

size = 5000

a_cpu = torch.randn(size, size)
b_cpu = torch.randn(size, size)

start = time.time()
c_cpu = a_cpu @ b_cpu
cpu_time = time.time() - start
print(f"CPU: {cpu_time:.3f}s")

if torch.cuda.is_available():
    a_gpu = a_cpu.to("cuda")
    b_gpu = b_cpu.to("cuda")

    torch.cuda.synchronize()
    start = time.time()
    c_gpu = a_gpu @ b_gpu
    torch.cuda.synchronize()
    gpu_time = time.time() - start
    print(f"GPU: {gpu_time:.3f}s")
    print(f"Speedup: {cpu_time / gpu_time:.0f}x")
```

## 练习

1. 运行上面的基准测试并比较 CPU 与 GPU 时间
2.如果没有GPU，可以在Google Colab上运行一下并进行比较
3. 检查您有多少 GPU 内存并估计您可以容纳的最大模型（经验法则：对于 fp16，每个参数 2 个字节）

## 关键术语

|术语 |人们怎么说|它实际上意味着什么 |
|------|----------------|----------------------|
| CUDA | 《GPU 编程》| NVIDIA 的并行计算平台可让您在 GPU 上运行代码 |
|显存 | “GPU内存” | GPU 上的视频 RAM，与系统 RAM 分开。限制模型尺寸。 |
| FP16 | “半精度”| 16 位浮点，使用 fp32 一半的内存，精度损失最小 |
|张量核心 | “快速矩阵硬件”|用于矩阵乘法的专用 GPU 内核，比常规内核快 4-8 倍 |
