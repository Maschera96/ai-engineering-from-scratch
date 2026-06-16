---
name: skill-jax-patterns-zh
description: Functional programming patterns 在 JAX -- 当 和 如何 到 使用 grad, jit, vmap, 和 pmap
version: 1.0.0
phase: 3
lesson: 12
tags: [jax, functional-programming, autodiff, compilation, vectorization]
---

# JAX Functional Patterns

JAX transforms pure 函数. Every pattern below follows one 规则: write a 函数 that takes 输入 和 returns 输出, 用 没有 side effects. Then transform it.

## Four Transforms

### grad -- Differentiate a 函数

```python
grads = jax.grad(loss_fn)(params, x, y)
loss, grads = jax.value_and_grad(loss_fn)(params, x, y)
```

使用 当: 你 need 梯度s 用于 optimization.
Constraint: 函数 must 返回 a scalar. For non-scalar 输出, 使用`jax.jacobian`.

### jit -- Compile a 函数

```python
fast_fn = jax.jit(f)
```

使用 当: 函数 will be called more than once 用 same-shaped 输入.
Constraint: 没有 Python control flow that depends 在 traced 值. 使用`jax.lax.cond`用于 conditionals,`jax.lax.scan`用于 loops.

### vmap -- Vectorize a 函数

```python
batch_fn = jax.vmap(f, in_axes=(None, 0))
```

使用 当: 你 wrote a 函数 用于 one 示例 和 need it 到 work 在 batches.
`in_axes`specifies which argument axis 到 批次 over.`None`means do 不 批次 (broadcast).

### pmap -- Parallelize across devices

```python
parallel_fn = jax.pmap(f, axis_name='devices')
```

使用 当: 你 have multiple GPUs/TPUs 和 want 数据 parallelism.
Inside 函数,`jax.lax.pmean(x, 'devices')`averages across devices.

## Composition Rules

Transforms compose. order matters:

```python
per_example_grads = jax.jit(jax.vmap(jax.grad(loss_fn), in_axes=(None, 0, 0)))
```

Reading right 到 left: take 梯度 of 损失_fn, vectorize over 示例, compile result.

Valid compositions:
- `jit(grad(f))`-- compiled 梯度 computation
- `jit(vmap(f))`-- compiled batched computation
- `vmap(grad(f))`-- per-示例 梯度s
- `pmap(jit(f))`-- parallel compiled computation
- `grad(jit(f))`-- 梯度 of compiled 函数 (same as jit(grad(f)))

## Parameter Management Pattern

JAX 参数 是 pytrees (nested dicts of arrays):

```python
params = {
    'layer1': {'w': jnp.zeros((784, 256)), 'b': jnp.zeros(256)},
    'layer2': {'w': jnp.zeros((256, 10)),  'b': jnp.zeros(10)},
}
```

Update all 参数 at once:
```python
params = jax.tree.map(lambda p, g: p - lr * g, params, grads)
```

Count 参数:
```python
n_params = sum(p.size for p in jax.tree.leaves(params))
```

## PRNG Key Management

JAX requires explicit random keys:

```python
key = jax.random.PRNGKey(0)
key, subkey = jax.random.split(key)
noise = jax.random.normal(subkey, shape)
```

For multiple random operations, split once:
```python
keys = jax.random.split(key, n)
```

绝不 reuse a key. 始终 split 之前 using.

## Common Mistakes

1. **Mutating arrays inside jit**: JAX arrays 是 immutable. 使用`x.at[i].set(v)`instead of`x[i] = v`.

2. **Using Python 打印 inside jit**:`print`runs during tracing, 不 execution. 使用`jax.debug.print("{}", x)`.

3. **Python 如果/用于 inside jit 在 traced 值**: 使用`jax.lax.cond`,`jax.lax.switch`,`jax.lax.scan`,`jax.lax.fori_loop`.

4. **Forgetting`.block_until_ready()`**: JAX uses async dispatch. For benchmarking, call`.block_until_ready()`到 wait 用于 actual completion.

5. **Reusing PRNG keys**: Two operations 用 same key produce same "random" 值. 始终 split.

6. **Global state 在 jitted 函数**: Global variables 是 captured at trace time. Changes 之后 tracing 是 invisible. Pass everything as arguments.

## Decision Checklist

1. Is 函数 called more than once? 加入`@jax.jit`.
2. Does it need 梯度s? Wrap 用`jax.grad`或`jax.value_and_grad`.
3. Does it process one 示例 but 你 have a 批次? Wrap 用`jax.vmap`.
4. Do 你 have multiple devices? Wrap 用`jax.pmap`.
5. Does it 使用 randomness? Thread PRNG keys through explicitly.
6. Does it have Python control flow 在 array 值? Replace 用`jax.lax`primitives.

## When 到 使用 JAX

使用 JAX 当:
- 你 need per-示例 梯度s (differential privacy, Fisher information)
- 你 是 训练 在 TPUs (JAX 是 native 框架)
- 你 need higher-order derivatives (Hessians, Jacobians)
- 你 want 到 compile entire 训练 步骤 到 a single kernel
- Your team 是 at Google DeepMind 或 Anthropic

使用 PyTorch 当:
- 你 want largest ecosystem (HuggingFace, torchvision, Lightning)
- 你 prioritize debugging ease over raw speed
- 你 是 deploying 到 NVIDIA GPUs 用 TorchServe/Triton
- 你 是 hiring (more PyTorch developers exist)
- 你 want 到 iterate fast 在 new architectures
