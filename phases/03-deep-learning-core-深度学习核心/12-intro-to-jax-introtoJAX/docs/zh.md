# JAX 入门

> PyTorch mutates 张量. TensorFlow builds graphs. JAX compiles pure 函数. That last one changes 如何 你 think about deep learning.

**Type:** 构建
**Languages:** Python
**Prerequisites:** Phase 03 Lessons 01-10, basic NumPy
**Time:** ~90 minutes

## 学习目标

- Write pure-函数 神经网络 code using JAX's functional API (jax.numpy, jax.grad, jax.jit, jax.vmap)
- 解释 key design difference between PyTorch's eager mutation 和 JAX's functional compilation 模型
- Apply jit compilation 和 vmap vectorization 到 accelerate 训练循环s compared 到 naive Python
- 训练 a 简单 network 在 JAX 和 contrast explicit state management 用 PyTorch's object-oriented approach

## 问题

你 know 如何 到 构建 神经网络 在 PyTorch. 你 define an`nn.Module`, call`.backward()`, 步骤 优化器. It works. Millions of people 使用 it.

But PyTorch has a constraint baked into its DNA: it traces operations eagerly, one at a time, 在 Python. Every`tensor + tensor`是 a separate kernel launch. Every 训练 步骤 re-interprets same Python code. 这 works fine until 你 need 到 训练 a 540-billion-parameter 模型 across 2,048 TPUs. Then overhead kills 你.

Google DeepMind trains Gemini 在 JAX. Anthropic trained Claude 在 JAX. 这些 是 不 small operations -- they 是 largest 神经网络 训练 runs 在 Earth. They chose JAX 因为 it treats 你的 训练循环 as a compilable program, 不 a sequence of Python calls.

JAX 是 NumPy 用 three superpowers: automatic differentiation, JIT compilation 到 XLA, 和 automatic vectorization. 你 write a 函数 that processes one 示例. JAX gives 你 a 函数 that processes a 批次, computes 梯度s, compiles 到 machine code, 和 runs across multiple devices. All 不用 changing original 函数.

## 概念

### JAX Philosophy

JAX 是 a functional 框架. No classes, 没有 mutable state, 没有`.backward()`method. Instead:

|PyTorch|JAX|
|---------|-----|
|`nn.Module`class 用 state|Pure 函数:`f(params, x) -> y`|
|`loss.backward()`|`jax.grad(loss_fn)(params, x, y)`|
|Eager execution|JIT compilation via XLA|
|`for x in batch:`manual loop|`jax.vmap(f)`auto-vectorization|
|`DataParallel`/`FSDP`|`jax.pmap(f)`auto-parallelism|
|Mutable`model.parameters()`|Immutable pytree of arrays|

这 是 不 a style preference. It 是 a compiler constraint. JIT compilation requires pure 函数 -- same 输入 always produce same 输出, 没有 side effects. That restriction 是 what makes 100x speedups possible.

### jax.numpy: Familiar Surface

JAX reimplements NumPy API 在 accelerators:

```python
import jax.numpy as jnp

a = jnp.array([1.0, 2.0, 3.0])
b = jnp.array([4.0, 5.0, 6.0])
c = jnp.dot(a, b)
```

Same 函数 names. Same broadcasting 规则. Same slicing semantics. But arrays live 在 GPU/TPU, 和 every operation 是 traceable by compiler.

One critical difference: JAX arrays 是 immutable. No`a[0] = 5`. Instead:`a = a.at[0].set(5)`. 这 feels awkward 用于 a week, 然后 it clicks -- immutability 是 what makes transformations like`grad`,`jit`, 和`vmap`composable.

### jax.grad: Functional Autodiff

PyTorch attaches 梯度s 到 张量 (`.grad`). JAX attaches 梯度s 到 函数.

```python
import jax

def f(x):
    return x ** 2

df = jax.grad(f)
df(3.0)
```

`jax.grad`takes a 函数 和 returns a new 函数 that computes 梯度. No`.backward()`call. No computation graph stored 在 张量. 梯度 是 just another 函数 你 can call, compose, 或 JIT-compile.

这 composes arbitrarily:

```python
d2f = jax.grad(jax.grad(f))
d2f(3.0)
```

Second derivatives. Third derivatives. Jacobians. Hessians. All by composing`grad`. PyTorch can do 这 too (`torch.autograd.functional.hessian`), but it 是 bolted 在. In JAX, it 是 foundation.

constraint:`grad`only works 在 pure 函数. No 打印 statements inside (they 运行 during tracing, 不 execution). No mutation of external state. No random number generation 不用 explicit key management.

### jit: Compile 到 XLA

```python
@jax.jit
def train_step(params, x, y):
    loss = loss_fn(params, x, y)
    return loss

fast_step = jax.jit(train_step)
```

On first call, JAX traces 函数 -- it records which operations happen, 不用 executing them. Then it hands that trace 到 XLA (Accelerated Linear Algebra), Google's compiler 用于 TPUs 和 GPUs. XLA fuses operations, eliminates redundant 内存 copies, 和 generates optimized machine code.

Subsequent calls skip Python entirely. compiled code runs 在 accelerator at C++ speed.

When JIT helps:
- 训练 步骤 (same computation repeated thousands of times)
- Inference (same 模型, different 输入)
- Any 函数 called more than once 用 similar-shaped 输入

When JIT hurts:
- Functions 用 Python control flow that depends 在 值 (`if x > 0`其中 x 是 a traced array)
- One-shot computations (compilation overhead exceeds runtime)
- Debugging (tracing hides actual execution)

control flow restriction 是 real.`jax.lax.cond`replaces`if/else`.`jax.lax.scan`replaces`for`loops. 这些 是 不 optional -- they 是 price of compilation.

### vmap: Automatic Vectorization

你 write a 函数 that processes one 示例:

```python
def predict(params, x):
    return jnp.dot(params['w'], x) + params['b']
```

`vmap`lifts it 到 process a 批次:

```python
batch_predict = jax.vmap(predict, in_axes=(None, 0))
```

`in_axes=(None, 0)`means: do 不 批次 over`params`(shared), 批次 over axis 0 of`x`. No manual`for`loop. No reshaping. No 批次 dimension threading. JAX figures out 批次 dimension 和 vectorizes entire computation.

这 是 不 syntactic sugar.`vmap`generates fused vectorized code that runs 10-100x faster than a Python loop. And it composes 用`jit`和`grad`:

```python
per_example_grads = jax.vmap(jax.grad(loss_fn), in_axes=(None, 0, 0))
```

Per-示例 梯度s. One line. 这 是 nearly impossible 在 PyTorch 不用 hacks.

### pmap: 数据 Parallelism Across Devices

```python
parallel_step = jax.pmap(train_step, axis_name='devices')
```

`pmap`replicates 函数 across all available devices (GPUs/TPUs) 和 splits 批次. Inside 函数,`jax.lax.pmean`和`jax.lax.psum`synchronize 梯度s across devices.

Google trains Gemini across thousands of TPU v5e chips using`pmap`(和 its successor`shard_map`). programming 模型: write single-设备 version, wrap 用`pmap`, done.

### Pytrees: Universal 数据 Structure

JAX operates 在 "pytrees" -- nested combinations of lists, tuples, dicts, 和 arrays. Your 模型 参数 是 a pytree:

```python
params = {
    'layer1': {'w': jnp.zeros((784, 256)), 'b': jnp.zeros(256)},
    'layer2': {'w': jnp.zeros((256, 128)), 'b': jnp.zeros(128)},
    'layer3': {'w': jnp.zeros((128, 10)),  'b': jnp.zeros(10)},
}
```

Every JAX transformation --`grad`,`jit`,`vmap`-- knows 如何 到 traverse pytrees.`jax.tree.map(f, tree)`applies`f`到 every leaf. 这 是 如何 优化器 update all 参数 at once:

```python
params = jax.tree.map(lambda p, g: p - lr * g, params, grads)
```

No`.parameters()`method. No parameter registration. tree structure 是 模型.

### Functional vs Object-Oriented

PyTorch stores state inside objects:

```python
class Model(nn.Module):
    def __init__(self):
        self.linear = nn.Linear(784, 10)

    def forward(self, x):
        return self.linear(x)
```

JAX uses pure 函数 用 explicit state:

```python
def predict(params, x):
    return jnp.dot(x, params['w']) + params['b']
```

params 是 passed 在. Nothing 是 stored. Nothing 是 mutated. 这 makes every 函数 testable, composable, 和 compilable. It also means 你 manage params yourself -- 或 使用 a library like Flax 或 Equinox.

### JAX Ecosystem

JAX gives 你 primitives. Libraries give 你 ergonomics:

|Library|Role|Style|
|---------|------|-------|
|**Flax** (Google)|Neural network 层|`nn.Module`用 explicit state|
|**Equinox** (Patrick Kidger)|Neural network 层|Pytree-based, Pythonic|
|**Optax** (DeepMind)|优化器 + LR schedules|Composable 梯度 transforms|
|**Orbax** (Google)|Checkpointing|Save/restore pytrees|
|**CLU** (Google)|Metrics + logging|训练 loop utilities|

Optax 是 standard 优化器 library. It separates 梯度 transformation (Adam, SGD, clipping) 从 parameter update, making it trivial 到 compose:

```python
optimizer = optax.chain(
    optax.clip_by_global_norm(1.0),
    optax.adam(learning_rate=1e-3),
)
```

### When 到 使用 JAX vs PyTorch

|Factor|JAX|PyTorch|
|--------|-----|---------|
|TPU support|First-class (Google built both)|Community-maintained (torch_xla)|
|GPU support|Good (CUDA via XLA)|Best-在-class (native CUDA)|
|Debugging|Hard (tracing + compilation)|Easy (eager, line-by-line)|
|Ecosystem|Research-focused (Flax, Equinox)|Massive (HuggingFace, torchvision, etc.)|
|Hiring|Niche (Google/DeepMind/Anthropic)|Mainstream (everywhere)|
|Large-尺度 训练|Superior (XLA, pmap, mesh)|Good (FSDP, DeepSpeed)|
|Prototyping speed|Slower (functional overhead)|Faster (mutate 和 go)|
|Production 推理|TensorFlow Serving, Vertex AI|TorchServe, Triton, ONNX|
|Who uses it|DeepMind (Gemini), Anthropic (Claude)|Meta (Llama), OpenAI (GPT), Stability AI|

honest answer: 使用 PyTorch unless 你 have a specific reason 到 使用 JAX. Those reasons 是 -- TPU access, need 用于 per-示例 梯度s, multi-设备 训练 at massive 尺度, 或 working at Google/DeepMind/Anthropic.

### Random Numbers 在 JAX

JAX does 不 have a global random state. Every random operation requires an explicit PRNG key:

```python
key = jax.random.PRNGKey(42)
key1, key2 = jax.random.split(key)
w = jax.random.normal(key1, shape=(784, 256))
```

这 是 annoying at first. But it guarantees reproducibility across devices 和 compilations -- a property that PyTorch's`torch.manual_seed`cannot guarantee 在 multi-GPU settings.

```figure
batchnorm-effect
```

## 动手构建

### Step 1: Setup 和 数据

We will 训练 a 3-层 MLP 在 MNIST using JAX 和 Optax. 784 输入, two hidden 层 of 256 和 128 neurons, 10 输出 classes.

```python
import jax
import jax.numpy as jnp
from jax import random
import optax

def get_mnist_data():
    from sklearn.datasets import fetch_openml
    mnist = fetch_openml('mnist_784', version=1, as_frame=False, parser='auto')
    X = mnist.data.astype('float32') / 255.0
    y = mnist.target.astype('int')
    X_train, X_test = X[:60000], X[60000:]
    y_train, y_test = y[:60000], y[60000:]
    return X_train, y_train, X_test, y_test
```

### Step 2: Initialize 参数

No class. Just a 函数 that returns a pytree:

```python
def init_params(key):
    k1, k2, k3 = random.split(key, 3)
    scale1 = jnp.sqrt(2.0 / 784)
    scale2 = jnp.sqrt(2.0 / 256)
    scale3 = jnp.sqrt(2.0 / 128)
    params = {
        'layer1': {
            'w': scale1 * random.normal(k1, (784, 256)),
            'b': jnp.zeros(256),
        },
        'layer2': {
            'w': scale2 * random.normal(k2, (256, 128)),
            'b': jnp.zeros(128),
        },
        'layer3': {
            'w': scale3 * random.normal(k3, (128, 10)),
            'b': jnp.zeros(10),
        },
    }
    return params
```

He-initialization, done manually. Three PRNG keys split 从 one seed. Every weight 是 an immutable array 在 a nested dict.

### Step 3: 前向传播

```python
def forward(params, x):
    x = jnp.dot(x, params['layer1']['w']) + params['layer1']['b']
    x = jax.nn.relu(x)
    x = jnp.dot(x, params['layer2']['w']) + params['layer2']['b']
    x = jax.nn.relu(x)
    x = jnp.dot(x, params['layer3']['w']) + params['layer3']['b']
    return x

def loss_fn(params, x, y):
    logits = forward(params, x)
    one_hot = jax.nn.one_hot(y, 10)
    return -jnp.mean(jnp.sum(jax.nn.log_softmax(logits) * one_hot, axis=-1))
```

Pure 函数. Params 在, 预测 out. No`self`, 没有 stored state.`loss_fn`computes 交叉熵 从零实现 -- softmax, log, negative 均值.

### Step 4: JIT-Compiled 训练 Step

```python
@jax.jit
def train_step(params, opt_state, x, y):
    loss, grads = jax.value_and_grad(loss_fn)(params, x, y)
    updates, opt_state = optimizer.update(grads, opt_state, params)
    params = optax.apply_updates(params, updates)
    return params, opt_state, loss

@jax.jit
def accuracy(params, x, y):
    logits = forward(params, x)
    preds = jnp.argmax(logits, axis=-1)
    return jnp.mean(preds == y)
```

`jax.value_and_grad`returns both 损失 值 和 梯度s 在 one pass.`@jax.jit`decorator compiles both 函数 到 XLA. After first call, each 训练 步骤 runs 不用 touching Python.

### Step 5: 训练循环

```python
optimizer = optax.adam(learning_rate=1e-3)

X_train, y_train, X_test, y_test = get_mnist_data()
X_train, X_test = jnp.array(X_train), jnp.array(X_test)
y_train, y_test = jnp.array(y_train), jnp.array(y_test)

key = random.PRNGKey(0)
params = init_params(key)
opt_state = optimizer.init(params)

batch_size = 128
n_epochs = 10

for epoch in range(n_epochs):
    key, subkey = random.split(key)
    perm = random.permutation(subkey, len(X_train))
    X_shuffled = X_train[perm]
    y_shuffled = y_train[perm]

    epoch_loss = 0.0
    n_batches = len(X_train) // batch_size
    for i in range(n_batches):
        start = i * batch_size
        xb = X_shuffled[start:start + batch_size]
        yb = y_shuffled[start:start + batch_size]
        params, opt_state, loss = train_step(params, opt_state, xb, yb)
        epoch_loss += loss

    train_acc = accuracy(params, X_train[:5000], y_train[:5000])
    test_acc = accuracy(params, X_test, y_test)
    print(f"Epoch {epoch + 1:2d} | Loss: {epoch_loss / n_batches:.4f} | "
          f"Train Acc: {train_acc:.4f} | Test Acc: {test_acc:.4f}")
```

10 轮次. ~97% test 准确率. first 轮次 是 slow (JIT compilation). Epochs 2-10 是 fast.

Notice what 是 missing: 没有`.zero_grad()`, 没有`.backward()`, 没有`.step()`. entire update 是 one composed 函数 call. 梯度s 是 computed, transformed by Adam, 和 applied 到 参数 -- all inside`train_step`.

## 直接使用

### Flax: Google Standard

Flax 是 most common JAX 神经网络 library. It adds`nn.Module`back, but 用 explicit state management:

```python
import flax.linen as nn

class MLP(nn.Module):
    @nn.compact
    def __call__(self, x):
        x = nn.Dense(256)(x)
        x = nn.relu(x)
        x = nn.Dense(128)(x)
        x = nn.relu(x)
        x = nn.Dense(10)(x)
        return x

model = MLP()
params = model.init(jax.random.PRNGKey(0), jnp.ones((1, 784)))
logits = model.apply(params, x_batch)
```

Same structure as PyTorch, but`params`是 separate 从 模型.`model.init()`creates params.`model.apply(params, x)`runs 前向传播. 模型 object has 没有 state.

### Equinox: Pythonic Alternative

Equinox (by Patrick Kidger) represents 模型s as pytrees:

```python
import equinox as eqx

model = eqx.nn.MLP(
    in_size=784, out_size=10, width_size=256, depth=2,
    activation=jax.nn.relu, key=jax.random.PRNGKey(0)
)
logits = model(x)
```

模型 itself 是 a pytree. No`.apply()`needed. 参数 是 just 模型's leaves. 这 是 closer 到 如何 JAX thinks.

### Optax: Composable 优化器

Optax decouples 梯度 transformation 从 update:

```python
schedule = optax.warmup_cosine_decay_schedule(
    init_value=0.0, peak_value=1e-3,
    warmup_steps=1000, decay_steps=50000
)

optimizer = optax.chain(
    optax.clip_by_global_norm(1.0),
    optax.adamw(learning_rate=schedule, weight_decay=0.01),
)
```

梯度 clipping, 学习率 warmup, 权重衰减 -- all composed as a chain of transforms. Each transform sees 梯度s, modifies them, 和 passes them 到 next. No monolithic 优化器 class.

## 交付它

**Installation:**

```bash
pip install jax jaxlib optax flax
```

For GPU support:

```bash
pip install jax[cuda12]
```

For TPU (Google Cloud):

```bash
pip install jax[tpu] -f https://storage.googleapis.com/jax-releases/libtpu_releases.html
```

**Performance gotchas:**

- First JIT call 是 slow (compilation). Warm up 之前 benchmarking.
- Avoid Python loops over JAX arrays inside JIT. 使用`jax.lax.scan`或`jax.lax.fori_loop`.
- `jax.debug.print()`works inside JIT. Regular`print()`does 不.
- Profile 用`jax.profiler`或 TensorBoard. XLA compilation can hide bottlenecks.
- JAX pre-allocates 75% of GPU 内存 by 默认. Set`XLA_PYTHON_CLIENT_PREALLOCATE=false`到 disable.

**Checkpointing:**

```python
import orbax.checkpoint as ocp
checkpointer = ocp.PyTreeCheckpointer()
checkpointer.save('/tmp/model', params)
restored = checkpointer.restore('/tmp/model')
```

**This lesson produces:**
- `outputs/prompt-jax-optimizer.md`-- a prompt 用于 choosing right JAX 优化器 configuration
- `outputs/skill-jax-patterns.md`-- a skill covering functional patterns 在 JAX

## Exercises

1. 加入 dropout 到 MLP. In JAX, dropout requires a PRNG key -- thread a key through 前向传播 和 split it 用于 each dropout 层. 比较 test 准确率 用 和 不用.

2. 使用`jax.vmap`到 compute per-示例 梯度s 用于 a 批次 of 32 MNIST images. Compute 梯度 norm 用于 each 示例. Which 示例 have largest 梯度s, 和 为什么?

3. Replace manual forward 函数 用 a generic`mlp_forward(params, x)`that works 用于 any number of 层. 使用`jax.tree.leaves`到 determine depth automatically.

4. Benchmark 训练 步骤 用 和 不用`@jax.jit`. Time 100 步骤 of each. How large 是 speedup 在 你的 hardware? What 是 compilation overhead 在 first call?

5. 实现 梯度 clipping by composing`optax.chain(optax.clip_by_global_norm(1.0), optax.adam(1e-3))`. 训练 用 和 不用 clipping. Plot 梯度 norm over 训练 到 see effect.

## Key Terms

|Term|What people say|What it actually means|
|------|----------------|----------------------|
|XLA|" thing that makes JAX fast"|Accelerated Linear Algebra -- a compiler that fuses operations 和 generates optimized GPU/TPU kernels 从 a computation graph|
|JIT|"Just-在-time compilation"|JAX traces 函数 在 first call, compiles 到 XLA, 然后 runs compiled version 在 subsequent calls|
|Pure 函数|"No side effects"|A 函数 其中 输出 depends only 在 输入 -- 没有 global state, 没有 mutation, 没有 randomness 不用 explicit keys|
|vmap|"Auto-batching"|Transforms a 函数 that processes one 示例 into one that processes a 批次, 不用 rewriting|
|pmap|"Auto-parallelism"|Replicates a 函数 across multiple devices 和 splits 输入 批次|
|Pytree|"Nested dict of arrays"|Any nested structure of lists, tuples, dicts, 和 arrays that JAX can traverse 和 transform|
|Tracing|"Recording computation"|JAX executes 函数 用 abstract 值 到 构建 a computation graph, 不用 computing real results|
|Functional autodiff|"grad of a 函数"|Computing derivatives by transforming 函数, 不 by attaching 梯度 storage 到 张量|
|Optax|"JAX's 优化器 library"|A composable library of 梯度 transformations -- Adam, SGD, clipping, scheduling -- that chain together|
|Flax|"JAX's nn.Module"|Google's 神经网络 library 用于 JAX, adding 层 abstractions while keeping state explicit|

## Further Reading

- JAX documentation:https://jax.readthedocs.io/-- official docs, 用 excellent tutorials 在 grad, jit, 和 vmap
- "JAX: composable transformations of Python+NumPy programs" (Bradbury et al., 2018) -- original paper explaining design philosophy
- Flax documentation:https://flax.readthedocs.io/-- Google's 神经网络 library 用于 JAX
- Patrick Kidger, "Equinox: 神经网络 在 JAX via callable PyTrees 和 filtered transformations" (2021) -- Pythonic alternative 到 Flax
- DeepMind, "Optax: composable 梯度 transformation 和 optimisation" -- standard 优化器 library
- "你 Don't Know JAX" (Colin Raffel, 2020) -- a practical guide 到 JAX gotchas 和 patterns, 从 one of T5 authors
