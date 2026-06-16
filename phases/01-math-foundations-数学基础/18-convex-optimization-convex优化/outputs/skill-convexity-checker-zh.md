---
name: skill-convexity-checker-zh
description: Determine if an 优化 problem is 凸 和 choose the right 求解器
version: 1.0.0
phase: 1
lesson: 18
tags: [optimization, convexity, solvers]
---

# Convexity Checker

How to verify whether an 优化 problem is 凸, 和 什么to do 与 the answer.

## Decision Checklist

1. Is the objective 函数 凸? (Check Hessian 正 semi-definiteness 或 use composition rules.)
2. Are all inequality 约束 of the form g_i(x) <= 0 where each g_i is 凸?
3. Are all equality 约束 affine (linear)?
4. If all three are yes, the problem is 凸. Use a 凸 求解器 与 convergence guarantees.
5. If any are no, the problem is non-凸. Use SGD/Adam 和 accept local optima.

## How to test convexity of a 函数

| Test | Applies to | Method |
|---|---|---|
| Second 导数 >= 0 | Scalar 函数 f(x) | Compute f''(x). If f''(x) >= 0 for all x, 凸. |
| Hessian is PSD | Multivariate 函数 f(x) | Compute H(x). If all 特征值 >= 0 everywhere, 凸. |
| Definition test | Any 函数 | Check f(tx + (1-t)y) <= t*f(x) + (1-t)*f(y) for sampled x, y, t. |
| Composition rules | Composed 函数 | See composition table below. |
| Restriction to a line | Multivariate f | f is 凸 iff g(t) = f(x + tv) is 凸 in t for all x, v. |

## Composition rules (preserving convexity)

| Operation | Result |
|---|---|
| f + g (both 凸) | 凸 |
| c * f (c > 0, f 凸) | 凸 |
| max(f, g) (both 凸) | 凸 |
| f(Ax + b) where f is 凸 | 凸 |
| g(f(x)) where g is 凸 non-decreasing 和 f is 凸 | 凸 |
| g(f(x)) where g is 凸 non-increasing 和 f is concave | 凸 |
| sum of 凸 函数 | 凸 |
| pointwise supremum of 凸 函数 | 凸 |

## Common ML objectives: 凸 或 not?

| Objective | 凸? | Reason |
|---|---|---|
| MSE: (1/n) sum(y - Xw)^2 | Yes | Quadratic in w, Hessian = (2/n) X^T X is PSD |
| Logistic 损失: sum(log(1 + exp(-y_i * w^T x_i))) | Yes | Sum of 凸 函数 (log-sum-exp family) |
| Hinge 损失: sum(max(0, 1 - y_i * w^T x_i)) | Yes | Max of 凸 (linear) 函数 |
| L2 regularization: lambda * \|\|w\|\|^2 | Yes | Quadratic, Hessian = 2*lambda*I |
| L1 regularization: lambda * \|\|w\|\|_1 | Yes | Sum of absolute values (凸 but not differentiable) |
| Ridge 回归: MSE + L2 | Yes | Sum of two 凸 函数 |
| LASSO: MSE + L1 | Yes | Sum of two 凸 函数 |
| Elastic net: MSE + L1 + L2 | Yes | Sum of 凸 函数 |
| SVM (primal): hinge + L2 | Yes | Sum of 凸 函数 |
| Cross-熵 与 softmax | Yes (in logits) | Log-sum-exp is 凸 |
| Neural network (any 损失) | No | Nonlinear activations create non-凸 composition |
| k-means objective | No | Discrete assignment step |
| 矩阵 factorization: \|\|X - UV^T\|\|^2 | No | Bilinear in U 和 V |
| GAN 损失 | No | Minimax, non-凸 in generator |
| Contrastive 损失 (InfoNCE) | No | Log of ratio of exponentials 与 负 样本 |

## Solver selection based on convexity

| Problem type | Solver | Convergence guarantee |
|---|---|---|
| 凸, smooth, unconstrained | 梯度 descent | O(1/k) to global minimum |
| 凸, smooth, unconstrained | L-BFGS | Superlinear to global minimum |
| 凸, smooth, unconstrained | Newton's method | Quadratic near minimum (if Hessian tractable) |
| 凸, smooth, constrained | Interior point method | Polynomial time |
| 凸, non-smooth (L1) | Proximal 梯度 / ISTA | O(1/k) to global minimum |
| 凸, non-smooth (L1) | ADMM | Flexible, h和les 约束 |
| 凸, quadratic | Conjugate 梯度 | Exact in n steps |
| Non-凸, smooth | SGD / Adam | Converges to local minimum |
| Non-凸, smooth | SGD + restarts | Better local minimum on average |
| Non-凸, smooth | Overparameterize + SGD | Flat minima, good generalization |

## Common mistakes

- Assuming a problem is 凸 because the 损失 函数 is 凸. The 损失 must be 凸 in the parameters you are optimizing. Cross-熵 is 凸 in the logits, but the full neural network mapping from 输入 to logits is non-凸.
- Using Newton's method on a non-凸 problem. The Hessian may have 负 特征值, causing Newton to move toward saddle points 或 maxima instead of minima.
- Forgetting that L1 regularization makes the objective non-differentiable at zero. St和ard 梯度 descent does not work well. Use proximal 梯度 descent 或 subgradient methods.
- Squaring the condition number by forming A^T A. If you need to 求解 a least-squares problem 和 A is ill-conditioned, use QR 或 SVD instead of the normal 方程.
- Declaring a problem non-凸 without checking. Many ML problems (linear 模型, SVMs, logistic 回归) are 凸 和 benefit from stronger solvers.

## Quick test: is my problem 凸?

```
1. Write out the objective: minimize f(w) subject to constraints
2. For each term in f(w):
   - Is it quadratic with PSD matrix? -> Convex
   - Is it a norm? -> Convex
   - Is it log-sum-exp? -> Convex
   - Does it involve w nonlinearly (sigmoid(w), w1*w2)? -> Likely non-convex
3. Are all constraints linear or convex inequalities?
4. If ALL terms are convex and constraints are convex/linear -> problem is convex
5. If ANY term is non-convex -> problem is non-convex
```
