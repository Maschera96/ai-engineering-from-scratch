# 概率和分布

> 概率是人工智能用来表达不确定性的语言。

**类型：** ** Learn
**语言：** Python
**先修：** ** 第 1 阶段，第 01-04 课
**时间：** ** 约 75 分钟

## 学习目标

- 从头开始实现伯努利分布、分类分布、泊松分布、均匀分布和正态分布的 PMF 和 PDF
- 计算期望值、方差，并使用中心极限定理来解释为什么高斯占主导地位
- 使用数值稳定性技巧构建 softmax 和 log-softmax 函数（减去最大 logit）
- 从 logits 计算交叉熵损失并将其连接到负对数似然

＃＃ 问题

分类器输出`[0.03, 0.91, 0.06]`。语言模型从 50,000 个候选词中选择下一个单词。扩散模型通过从学习的分布中采样来生成图像。所有这些都是行动中的概率。

模型做出的每个预测都是概率分布。每个损失函数都会衡量预测分布与真实分布的差距。每个训练步骤都会调整参数以使一个分布看起来更像另一个分布。如果没有概率，您就无法阅读一篇 ML 论文、调试单个模型或理解为什么您的训练损失为 NaN。

## 概念

### 事件、样本空间和概率

样本空间S是所有可能结果的集合。事件是样本空间的子集。概率将事件映射到 0 到 1 之间的数字。

```
Coin flip:
  S = {H, T}
  P(H) = 0.5,  P(T) = 0.5

Single die roll:
  S = {1, 2, 3, 4, 5, 6}
  P(even) = P({2, 4, 6}) = 3/6 = 0.5
```

三个公理定义了所有概率：
1. 对于任何事件 A，P(A) >= 0
2. P(S) = 1（总有事情发生）
3. 当 A 和 B 不能同时出现时，P(A 或 B) = P(A) + P(B)

其他一切（贝叶斯定理、期望、分布）都遵循这三个规则。

### 条件概率和独立性

P(A|B) 是在 B 发生的情况下 A 发生的概率。

```
P(A|B) = P(A and B) / P(B)

Example: deck of cards
  P(King | Face card) = P(King and Face card) / P(Face card)
                      = (4/52) / (12/52)
                      = 4/12 = 1/3
```

当知道一个事件不告诉你另一个事件时，两个事件是独立的：

```
Independent:   P(A|B) = P(A)
Equivalent to: P(A and B) = P(A) * P(B)
```

抛硬币是独立的。抽卡不更换则不然。

### 概率质量函数与概率密度函数

离散随机变量具有概率质量函数 (PMF)。每个结果都有一个特定的概率，您可以直接读出。

```
PMF: P(X = k)

Fair die:
  P(X = 1) = 1/6
  P(X = 2) = 1/6
  ...
  P(X = 6) = 1/6

  Sum of all probabilities = 1
```

连续随机变量具有概率密度函数 (PDF)。单点的密度不是概率。概率来自于对一段时间间隔内的密度进行积分。

```
PDF: f(x)

P(a <= X <= b) = integral of f(x) from a to b

f(x) can be greater than 1 (density, not probability)
integral from -inf to +inf of f(x) dx = 1
```

这种区别在机器学习中很重要。分类输出是 PMF（离散选择）。 VAE 潜在空间使用 PDF（连续）。

### 常见分布

**伯努利：** 一项试验，两种结果。模型二元分类。

```
P(X = 1) = p
P(X = 0) = 1 - p
Mean = p,  Variance = p(1-p)
```

**分类：** 一项试验，k 个结果。建立多类分类模型（softmax 输出）。

```
P(X = i) = p_i,  where sum of p_i = 1
Example: P(cat) = 0.7,  P(dog) = 0.2,  P(bird) = 0.1
```

**统一：**所有结果的可能性均等。用于随机初始化。

```
Discrete: P(X = k) = 1/n for k in {1, ..., n}
Continuous: f(x) = 1/(b-a) for x in [a, b]
```

**正态（高斯）：** 钟形曲线。通过均值 (mu) 和方差 (sigma^2) 进行参数化。

```
f(x) = (1 / sqrt(2*pi*sigma^2)) * exp(-(x - mu)^2 / (2*sigma^2))

Standard normal: mu = 0, sigma = 1
  68% of data within 1 sigma
  95% within 2 sigma
  99.7% within 3 sigma
```

**泊松：**固定间隔内罕见事件的计数。对事件发生率进行建模。

```
P(X = k) = (lambda^k * e^(-lambda)) / k!
Mean = lambda,  Variance = lambda
```

### 期望值和方差

期望值是加权平均结果。

```
Discrete:   E[X] = sum of x_i * P(X = x_i)
Continuous: E[X] = integral of x * f(x) dx
```

方差度量分布在均值附近。

```
Var(X) = E[(X - E[X])^2] = E[X^2] - (E[X])^2
Standard deviation = sqrt(Var(X))
```

在机器学习中，期望值显示为损失函数（数据分布的平均损失）。方差告诉您模型的稳定性。梯度的高方差意味着训练的噪音。

### 联合分布和边际分布

联合分布 P(X, Y) 一起描述两个随机变量。

联合 PMF 示例（X = 天气，Y = 雨伞）：

| | Y=0（无伞）| Y=1（伞）|边际 P(X) |
|---|---|---|---|
| X=0（太阳）| 0.40 | 0.40 0.10 | 0.10 P(X=0) = 0.50 | P(X=0) = 0.50
| X=1（雨）| 0.05 | 0.05 0.45 | 0.45 P(X=1) = 0.50 |
| **边际 P(Y)** | P(Y=0) = 0.45 | P(Y=0) = 0.45 P(Y=1) = 0.55 | P(Y=1) = 0.55 1.00 |

边际分布对其他变量求和：

```
P(X = x) = sum over all y of P(X = x, Y = y)
```

上表中的行和列总计是边际值。

### 为什么正态分布无处不在

中心极限定理：许多独立随机变量的总和（或平均值）收敛于正态分布，无论原始分布如何。

```
Roll 1 die:  uniform distribution (flat)
Average of 2 dice:  triangular (peaked)
Average of 30 dice: nearly perfect bell curve

This works for ANY starting distribution.
```

这就是为什么：
- 测量误差近似正常（许多小的独立来源）
- 神经网络中的权重初始化使用正态分布
- SGD 中的梯度噪声近似正常（许多样本梯度的总和）
- 正态分布是给定均值和方差的最大熵分布

### 对数概率

原始概率会导致数值问题。将许多小概率相乘很快就会下溢到零。

```
P(sentence) = P(word1) * P(word2) * ... * P(word_n)
            = 0.01 * 0.003 * 0.02 * ...
            -> 0.0 (underflow after ~30 terms)
```

对数概率可以解决这个问题。乘法变为加法。

```
log P(sentence) = log P(word1) + log P(word2) + ... + log P(word_n)
                = -4.6 + -5.8 + -3.9 + ...
                -> finite number (no underflow)
```

规则：
- log(a * b) = log(a) + log(b)
- 对数概率始终 <= 0（因为 0 < P <= 1）
- 更负面 = 可能性更小
- 交叉熵损失是正确类别的负对数概率

### Softmax 作为概率分布

神经网络输出原始分数（logits）。 Softmax 将它们转换为有效的概率分布。

```
softmax(z_i) = exp(z_i) / sum(exp(z_j) for all j)

Properties:
  - All outputs are in (0, 1)
  - All outputs sum to 1
  - Preserves relative ordering of inputs
  - exp() amplifies differences between logits
```

softmax 技巧：在求幂之前减去最大 logit 以防止溢出。

```
z = [100, 101, 102]
exp(102) = overflow

z_shifted = z - max(z) = [-2, -1, 0]
exp(0) = 1  (safe)

Same result, no overflow.
```

Log-softmax 结合了 softmax 和 log 以实现数值稳定性。 PyTorch 在内部使用它来进行交叉熵损失。

### 采样

抽样意味着从分布中抽取随机值。在机器学习中：
- Dropout 随机采样哪些神经元归零
- 数据增强样本随机变换
- 语言模型从预测分布中采样下一个标记
- 扩散模型对噪声进行采样并逐渐降噪

从任意分布中采样需要诸如逆变换采样、拒绝采样或重新参数化技巧（在 VAE 中使用）等技术。

```figure
gaussian-pdf
```

## Build It

### 第 1 步：概率基础知识

```python
import math
import random

def factorial(n):
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result

def combinations(n, k):
    return factorial(n) // (factorial(k) * factorial(n - k))

def conditional_probability(p_a_and_b, p_b):
    return p_a_and_b / p_b

p_king_given_face = conditional_probability(4/52, 12/52)
print(f"P(King | Face card) = {p_king_given_face:.4f}")
```

### 步骤 2：从头开始 PMF 和 PDF

```python
def bernoulli_pmf(k, p):
    return p if k == 1 else (1 - p)

def categorical_pmf(k, probs):
    return probs[k]

def poisson_pmf(k, lam):
    return (lam ** k) * math.exp(-lam) / factorial(k)

def uniform_pdf(x, a, b):
    if a <= x <= b:
        return 1.0 / (b - a)
    return 0.0

def normal_pdf(x, mu, sigma):
    coeff = 1.0 / (sigma * math.sqrt(2 * math.pi))
    exponent = -0.5 * ((x - mu) / sigma) ** 2
    return coeff * math.exp(exponent)
```

### 步骤 3：期望值和方差

```python
def expected_value(values, probabilities):
    return sum(v * p for v, p in zip(values, probabilities))

def variance(values, probabilities):
    mu = expected_value(values, probabilities)
    return sum(p * (v - mu) ** 2 for v, p in zip(values, probabilities))

die_values = [1, 2, 3, 4, 5, 6]
die_probs = [1/6] * 6
mu = expected_value(die_values, die_probs)
var = variance(die_values, die_probs)
print(f"Die: E[X] = {mu:.4f}, Var(X) = {var:.4f}, SD = {var**0.5:.4f}")
```

### 步骤 4：从分布中采样

```python
def sample_bernoulli(p, n=1):
    return [1 if random.random() < p else 0 for _ in range(n)]

def sample_categorical(probs, n=1):
    cumulative = []
    total = 0
    for p in probs:
        total += p
        cumulative.append(total)
    samples = []
    for _ in range(n):
        r = random.random()
        for i, c in enumerate(cumulative):
            if r <= c:
                samples.append(i)
                break
    return samples

def sample_normal_box_muller(mu, sigma, n=1):
    samples = []
    for _ in range(n):
        u1 = random.random()
        u2 = random.random()
        z = math.sqrt(-2 * math.log(u1)) * math.cos(2 * math.pi * u2)
        samples.append(mu + sigma * z)
    return samples
```

### 步骤 5：Softmax 和对数概率

```python
def softmax(logits):
    max_logit = max(logits)
    shifted = [z - max_logit for z in logits]
    exps = [math.exp(z) for z in shifted]
    total = sum(exps)
    return [e / total for e in exps]

def log_softmax(logits):
    max_logit = max(logits)
    shifted = [z - max_logit for z in logits]
    log_sum_exp = max_logit + math.log(sum(math.exp(z) for z in shifted))
    return [z - log_sum_exp for z in logits]

def cross_entropy_loss(logits, target_index):
    log_probs = log_softmax(logits)
    return -log_probs[target_index]
```

### 步骤 6：中心极限定理演示

```python
def demonstrate_clt(dist_fn, n_samples, n_averages):
    averages = []
    for _ in range(n_averages):
        samples = [dist_fn() for _ in range(n_samples)]
        averages.append(sum(samples) / len(samples))
    return averages
```

### 步骤 7：可视化

```python
import matplotlib.pyplot as plt

xs = [mu + sigma * (i - 500) / 100 for i in range(1001)]
ys = [normal_pdf(x, mu, sigma) for x, mu, sigma in ...]
plt.plot(xs, ys)
```

所有可视化的完整实现都在 `code/probability.py` 中。

## Use It

对于 NumPy 和 SciPy，上面的所有内容都只是一句台词：

```python
import numpy as np
from scipy import stats

normal = stats.norm(loc=0, scale=1)
samples = normal.rvs(size=10000)
print(f"Mean: {np.mean(samples):.4f}, Std: {np.std(samples):.4f}")
print(f"P(X < 1.96) = {normal.cdf(1.96):.4f}")

logits = np.array([2.0, 1.0, 0.1])
from scipy.special import softmax, log_softmax
probs = softmax(logits)
log_probs = log_softmax(logits)
print(f"Softmax: {probs}")
print(f"Log-softmax: {log_probs}")
```

您从头开始构建了这些。现在你知道图书馆的电话正在做什么了。

## 练习

1. 对指数分布实施逆变换采样。通过采样 10,000 个值并将直方图与真实 PDF 进行比较来进行验证。

2. 为两个加载的骰子建立联合分配表。计算边际分布并检查骰子是否独立。

3. 计算 5 类分类器的交叉熵损失，当正确的类是索引 3 时，该分类器输出 logits `[2.0, 0.5, -1.0, 3.0, 0.1]`。然后使用 PyTorch 的 `nn.CrossEntropyLoss` 验证您的答案。

4. 编写一个函数，获取对数概率列表并返回最可能的序列、总对数概率和等效的原始概率。使用包含 50 个单词的句子进行测试，其中每个单词的概率为 0.01。

## 关键术语

|术语 |人们怎么说|它实际上意味着什么 |
|------|----------------|----------------------|
|样本空间| “所有的可能性”|实验的每个可能结果的集合 S |
| PMF | “概率函数”|给出每个离散结果的精确概率的函数，总和为 1 |
| PDF | “概率曲线”|连续变量的密度函数。在一个区间内对其进行积分以获得概率 |
|条件概率| “给定某物的概率” | P(A\|B) = P(A 和 B) / P(B)。贝叶斯思维基础和贝叶斯定理|
|独立| “它们不会互相影响”| P(A 和 B) = P(A) * P(B)。了解一个事件并不能告诉你关于另一个事件的信息 |
|预期值| “平均”|所有结果的概率加权总和。损失函数是期望值 |
|方差 | “如何分散”|与平均值的预期平方偏差。高方差 = 嘈杂、不稳定的估计 |
|正态分布| “钟形曲线”| f(x) = (1/sqrt(2*pi*sigma^2)) * exp(-(x-mu)^2/(2*sigma^2))。由于 CLT 无处不在 |
|中心极限定理| “平均值变得正常”|无论来源如何，许多独立样本的均值都会收敛于正态分布 |
|联合配送| “两个变量在一起” | P(X, Y) 描述 X 和 Y 结果的每种组合的概率 |
|边际分布| “对其他变量求和”| P(X) = sum_y P(X, Y)。从联合 | 恢复一个变量的分布
|对数概率 | “概率的对数” |对数 P(x)。将乘积转化为和，防止长序列中的数值下溢 |
| Softmax| “将分数转化为概率” | softmax(z_i) = exp(z_i) / sum(exp(z_j))。将实值 logits 映射到有效的概率分布 |
|交叉熵 | “损失函数” | -sum(p_true * log(p_predicted))。衡量两个分布的差异程度。越低越好 |
|洛吉特| “原始模型输出”| softmax 之前的非标准化分数。以物流函数命名 |
|取样| “绘制随机值” |根据概率分布生成值。模型如何生成输出 |

## 延伸阅读

- [3Blue1Brown：但是什么是中心极限定理？](https://www.youtube.com/watch?v=zeJD6dqJ5lo) - 为什么平均值变得正常的视觉证明
- [斯坦福 CS229 概率评论](https://cs229.stanford.edu/section/cs229-prob.pdf) - 简明参考，涵盖此处的所有内容以及更多内容
- [Log-Sum-Exp 技巧](https://gregorygundersen.com/blog/2020/02/09/log-sum-exp/) - 为什么数值稳定性很重要以及如何实现它
