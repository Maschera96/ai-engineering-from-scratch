# 开发环境

> 工具会塑造你的思考方式。一次搭好，正确搭好。

**类型：** Build
**语言：** Python, Node.js, Rust
**先修：** 无
**时间：** 约 45 分钟

## 学习目标

- 从零搭建 Python 3.11+、Node.js 20+ 和 Rust 工具链
- 配置虚拟环境和包管理器，让构建过程可复现
- 使用 CUDA/MPS 验证 GPU 访问，并运行一次测试张量操作
- 理解四层栈：系统、包、运行时、AI 库

## 问题

你将用 Python、TypeScript、Rust 和 Julia 学习 200 多节 AI 工程课程。如果环境坏了，每一节课都会变成和工具链搏斗，而不是学习。

大多数人会跳过环境搭建。然后他们会花几个小时调试 import 错误、版本冲突和缺失的 CUDA 驱动。我们要把这件事认真做一次。

## 概念

AI 工程环境有四层：

```mermaid
graph TD
    A["4. AI/ML 库\nPyTorch, JAX, transformers, etc."] --> B["3. 语言运行时\nPython 3.11+, Node 20+, Rust, Julia"]
    B --> C["2. 包管理器\nuv, pnpm, cargo, juliaup"]
    C --> D["1. 系统基础\nOS, shell, git, editor, GPU drivers"]
```

我们从下往上安装。每一层都依赖下面那一层。

## 构建它

### 第 1 步：系统基础

检查你的系统并安装基础工具。

```bash
# macOS
xcode-select --install
brew install git curl wget

# Ubuntu/Debian
sudo apt update && sudo apt install -y build-essential git curl wget

# Windows (use WSL2)
wsl --install -d Ubuntu-24.04
```

### 第 2 步：使用 uv 安装 Python

我们使用 `uv`，它比 pip 快 10 到 100 倍，并且会自动处理虚拟环境。

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh

uv python install 3.12

uv venv
source .venv/bin/activate  # or .venv\Scripts\activate on Windows

uv pip install numpy matplotlib jupyter
```

验证：

```python
import sys
print(f"Python {sys.version}")

import numpy as np
print(f"NumPy {np.__version__}")
a = np.array([1, 2, 3])
print(f"Vector: {a}, dot product with itself: {np.dot(a, a)}")
```

### 第 3 步：使用 pnpm 安装 Node.js

用于 TypeScript 课程，包括智能体、MCP 服务器和 Web 应用。

```bash
curl -fsSL https://fnm.vercel.app/install | bash
fnm install 22
fnm use 22

npm install -g pnpm

node -e "console.log('Node', process.version)"
```

### 第 4 步：Rust

用于对性能敏感的课程，包括推理和系统。

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

rustc --version
cargo --version
```

### 第 5 步：Julia（可选）

用于 Julia 表现很好的重数学课程。

```bash
curl -fsSL https://install.julialang.org | sh

julia -e 'println("Julia ", VERSION)'
```

### 第 6 步：GPU 设置（如果你有 GPU）

```bash
# NVIDIA
nvidia-smi

# Install PyTorch with CUDA
uv pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu124
```

```python
import torch
print(f"CUDA available: {torch.cuda.is_available()}")
if torch.cuda.is_available():
    print(f"GPU: {torch.cuda.get_device_name(0)}")
```

没有 GPU？没问题。大多数课程都能在 CPU 上运行。对于训练较重的课程，可以使用 Google Colab 或云 GPU。

### 第 7 步：验证所有内容

运行验证脚本：

```bash
python phases/00-setup-and-tooling-环境搭建与工具链/01-dev-environment-开发环境/code/verify.py
```

## 使用它

你的环境现在已经可以支撑本课程的每一节课。下面是各语言的使用位置：

| 语言 | 使用位置 | 包管理器 |
|----------|---------|-----------------|
| Python | Phases 1-12（ML、DL、NLP、Vision、Audio、LLMs） | uv |
| TypeScript | Phases 13-17（Tools、Agents、Swarms、Infra） | pnpm |
| Rust | Phases 12, 15-17（对性能敏感的系统） | cargo |
| Julia | Phase 1（数学基础） | Pkg |

## 交付它

本课产出一个验证脚本，任何人都可以运行它来检查自己的环境。

查看 `outputs/prompt-env-check.md`，其中的提示词可以帮助 AI 助手诊断环境问题。

## 练习

1. 运行验证脚本并修复任何失败项
2. 为本课程创建一个 Python 虚拟环境并安装 PyTorch
3. 用四种语言分别写一个 "hello world"，并逐个运行
