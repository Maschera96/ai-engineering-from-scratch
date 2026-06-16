---
name: skill-gradient-computation-zh
description: Compute 梯度 of common ML 损失 函数 和 choose the right 导数 approach
version: 1.0.0
phase: 1
lesson: 4
tags: [calculus, gradients, backpropagation]
---

# 梯度 Computation for ML

Practical reference for computing 梯度 of 损失 函数, activation 函数, 和 layer operations used in neural networks.

## Decision Checklist

1. Is the 函数 composed of simple primitives (power, exp, log, trig)? Use analytical 导数 和 the chain rule.
2. Is the 函数 a custom 或 black-box operation? Use numerical differentiation: `(f(x+h) - f(x-h)) / (2h)` 与 h = 1e-7.
3. Is the 函数 built from 张量 operations in PyTorch/JAX? Let autograd h和le it. Verify 与 numerical check.
4. Do you need the 梯度 of a 标量 损失 w.r.t. a 矩阵 of weights? Apply the chain rule through the computation 图, one 节点 at a time.
5. Is there a non-differentiable operation (argmax, rounding, 采样)? Use a straight-through estimator 或 reparameterization trick.

## When to use each approach

| Approach | When to use | Cost |
|---|---|---|
| Analytical (h和-derived) | Simple 函数, verifying autograd 输出 | Free at runtime |
| Numerical (finite differences) | Debugging, 梯度 checking, black-box 函数 | 2n forward passes for n parameters |
| Automatic differentiation | Any differentiable computation 图 (the default) | One backward pass |
| Symbolic (SymPy, Mathematica) | Deriving closed-form 梯度 for papers | Compile time only |

## Quick reference: common 导数

| 函数 | f(x) | f'(x) | ML context |
|---|---|---|---|
| MSE 损失 | (1/n) sum(y_hat - y)^2 | (2/n)(y_hat - y) | Regression |
| Cross-熵 (binary) | -(y log(p) + (1-y) log(1-p)) | p - y (after sigmoid) | Binary 分类 |
| Cross-熵 (multi) | -log(p_true_class) | p - one_hot(y) (after softmax) | Multi-class 分类 |
| Sigmoid | 1 / (1 + e^(-x)) | sigma(x) * (1 - sigma(x)) | Output gates, binary 输出 |
| Tanh | (e^x - e^(-x)) / (e^x + e^(-x)) | 1 - tanh(x)^2 | Hidden activations (legacy) |
| ReLU | max(0, x) | 1 if x > 0, 0 if x < 0 | Default hidden activation |
| Leaky ReLU | max(0.01x, x) | 1 if x > 0, 0.01 if x < 0 | 避免ing dead neurons |
| GELU | x * Phi(x) | Phi(x) + x * phi(x) | Transformers |
| Softmax_i | e^(x_i) / sum(e^(x_j)) | s_i(1 - s_i) for i=j, -s_i*s_j for i!=j | Output layer (Jacobian) |
| Log-softmax | x_i - log(sum(e^(x_j))) | 1 - softmax(x_i) for the i-th entry | Numerically 稳定 CE |
| Linear layer | y = Wx + b | dL/dW = dL/dy * x^T, dL/db = dL/dy | Every layer |
| L2 regularization | lambda * sum(w^2) | 2 * lambda * w | Weight decay |
| L1 regularization | lambda * sum(\|w\|) | lambda * sign(w) | Sparsity |

## Common mistakes

- Forgetting the 1/n factor in batch-averaged losses (MSE, cross-熵). The 梯度 is scaled by batch size.
- Computing softmax 梯度 as a 向量 when it is actually a Jacobian 矩阵. For cross-熵 + softmax combined, the 梯度 simplifies to (p - y), which avoids the full Jacobian.
- Applying the chain rule in the wrong order. Work backward from the 损失: dL/dW = dL/dy * dy/dW.
- Using h that is too large (h = 0.1) 或 too small (h = 1e-15) for numerical 导数. Stick to h = 1e-7 for float64.
- Forgetting that ReLU has undefined 梯度 at exactly x = 0. In practice, set it to 0 或 0.5.

## 梯度 checking recipe

```
For each parameter w:
  numeric_grad = (loss(w + h) - loss(w - h)) / (2h)
  auto_grad = backward pass value
  relative_error = |numeric - auto| / max(|numeric|, |auto|, 1e-8)
  assert relative_error < 1e-5
```

Relative 误差 above 1e-3 means something is wrong. Between 1e-5 和 1e-3, investigate.
