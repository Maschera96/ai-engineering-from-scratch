# 推测解码 and EAGLE

> 一个frontier LLM generating one 词元 requires a full forward pass over billions of 参数. That forward pass is massively over-provisioned: most of the time a much smaller 模型 can guess the next 3-5 词元 correctly, and the big 模型 only needs to *verify* the guess. When the guess is right you got 5 词元 for the price of one. Speculative decoding (Leviathan et al. 2023) made this exact, and EAGLE-3 (2025) pushed acceptance rates to ~4.5 词元 per verify — a 4-5x speedup at matched 输出 分布.

**类型：** Build
**语言：** Python (with numpy)
**先修：** Phase 10 Lesson 12 (推理 优化), Phase 10 Lesson 04 (Pre-training Mini-GPT)
**时间：** 约 75 分钟

## 问题

Decode throughput for a 70B-class 模型 on H100 is typically 40-80 词元/second. Each 词元 requires a full forward pass reading all 模型 权重 from HBM. You cannot make the 模型 smaller without changing its 输出. You cannot increase 批次 size beyond 内存. You're stuck — unless you can let the 模型 输出 more than one 词元 per forward pass.

Autoregressive 生成 looks inherently serial: `x_{t+1} = sample(p(· | x_{1:t}))`. But there is a concurrency opportunity. If you had a cheap predictor that said "the next 4 词元 are probably [a, b, c, d]" you could verify all 5 positions in a **single forward pass of the big 模型** and accept the longest 匹配 prefix.

Leviathan, Kalai, Matias (2023, "Fast 推理 from Transformers via 推测解码") made this exact via a clever accept/reject rule that preserves the 目标 模型's 采样 分布. The same 输出 分布, 2-4× faster.

## 概念

### The Two-Model Setup

- **目标 模型** `M_p`: the big, slow, high-quality 模型 you actually want 样本 from. 分布: `p(x)`.
- **Draft 模型** `M_q`: a small, fast, lower-quality 模型. 分布: `q(x)`. 5-30× smaller.

Per 步骤:

1. Draft 模型 proposes `K` 词元 autoregressively: `x_1, x_2, ..., x_K ~ q`.
2. 目标 模型 runs ONE forward pass over all `K+1` positions in 并行, producing `p(x_k)` for each proposed 词元.
3. Accept/reject each 词元 left-to-right via the modified rejection-sampling rule below. Accept the longest 匹配 prefix.
4. 如果any 词元 is rejected, 样本 the replacement from the corrected 分布 and stop. Otherwise 样本 one bonus 词元 from `p(· | x_1...x_K)`.

如果the draft matches the 目标 perfectly, you get K+1 词元 per target-forward. If the draft is wrong at position 1, you get only 1 词元.

### The Exactness Rule

Speculative decoding is **provably equivalent in 分布 to 采样 from p**. The rejection rule:

```text
For each drafted token x_t:
    r ~ Uniform(0, 1)
    if r < p(x_t) / q(x_t):
        accept x_t
    else:
        sample replacement from residual: (p - q)+ / ||(p - q)+||_1
        stop
```

where `(p - q)+` denotes the positive part of the pointwise difference. When the draft and 目标 agree (`p ≈ q`) acceptance is nearly 1. When they disagree, the residual 分布 is constructed so that the overall 样本 is still exactly `p`.

**Greedy case.** For temperature=0 采样 just check `argmax(p) == x_t`. If yes, accept; if no, 输出 `argmax(p)` and stop.

### Expected Speedup

如果the draft 模型's token-level acceptance 速率 is `α`, the expected 词元 produced per target-forward pass is:

```text
E[tokens] = (1 - α^{K+1}) / (1 - α)        # K = draft length, α in [0, 1]
```

At `α = 0.8, K = 4`: `(1 - 0.8^5)/(1 - 0.8) = 3.36` 词元 per forward. A single 目标 forward 成本 roughly `cost_q * K + cost_p` (K draft 步骤 plus one 目标 verify). If `cost_p >> cost_q * K` the speedup 比例 is `3.36× / 1 = 3.36×` on throughput.

这个only 真实 参数 is `α`, which depends entirely on the draft-target 对齐. A good draft is everything.

### 训练 the Draft: Distillation

一个random small 模型 makes a poor draft. The standard recipe is to distill from the 目标:

1. 选择a small 架构 (~1B for a 70B 目标, ~500M for a 7B 目标).
2. 运行the 目标 模型 on a large 文本 语料库; store its next-token distributions.
3. 训练 the draft with KL divergence against the 目标's 分布 (not against ground-truth 词元).

这个result: `α` typically 0.6-0.8 on coding, 0.7-0.85 on natural-language chat. Speedups 2-3× in 生产.

### EAGLE: Tree Drafting + 特征 Reuse

Li, Wei, Zhang, Zhang (2024, "EAGLE: Speculative 采样 Requires Rethinking 特征 Uncertainty") observed two inefficiencies in standard 推测解码:

1. 这个draft does K serial 步骤, each full-stack. But the draft could reuse the 目标's 特征s (隐藏 states) from the most recent verify — the 目标 already computed rich representations that the draft is re-deriving from scratch.
2. 这个draft outputs a linear 链. If the draft could 输出 a *tree* of candidates (each 节点 multiple guesses), the 目标's single forward pass could verify multiple candidate paths in 并行 via a tree 注意力 掩码, and pick the longest accepted branch.

EAGLE-1 changes:
- Draft 输入 = 目标's final 隐藏 状态 at position t, not raw 词元.
- Draft 架构 = 1 transformer 解码器 层 (not a separate small 模型).
- 输出 = tree of K = 4-8 candidates per 深度, 深度 4-6.

EAGLE-2 (2024) adds dynamic tree 拓扑: the tree grows wider where the draft is uncertain and stays narrow where it is confident. Raises `α_effective` without increasing verify 成本.

EAGLE-3 (Li et al. 2025, "EAGLE-3: 扩展 up 推理 Acceleration of Large Language 模型 via Training-时间 Test") removes the fixed top-layer 特征 dependency and trains the draft with a new "test-time simulation" 损失 — the draft is 训练后的 on outputs that match the 目标's test-time 分布 rather than teacher-forced 训练 分布. Acceptance 速率 rises from 0.75 (EAGLE-2) to 0.82 (EAGLE-3), and mean 词元/verify from 3.0 to 4.5.

### Tree 注意力 Verification

当the draft outputs a tree, the 目标 模型 verifies it in a single forward pass using a **tree 注意力 掩码** — a 因果 掩码 that encodes the tree 拓扑 rather than a pure line. Each 词元 attends only to its ancestors in the tree. The verify pass is still one forward, one matmul; the topological 掩码 成本 only a few extra KV entries.

```text
        root
       /    \
      a      b
     / \    / \
    c  d   e   f
```

如果`a, b` are competing first-token candidates and `c, d, e, f` are second-token candidates, all six positions are verified in one forward pass. The 输出 is the longest prefix along any accepted path.

### When It Wins, When It Doesn't

**Wins:**
- Chat / completion with predictable 文本 (code, common English, 结构化 输出). `α` is high.
- Settings with unused GPU 计算 during decode (memory-bound phase). Tree drafting uses the available FLOPs.

**Loses / no win:**
- Highly stochastic outputs (creative writing at high temperature). `α` drops toward `1/|vocab|`.
- 批次 serving with very high concurrency — batching already fills the FLOPs, little room for tree verification.
- Very small 目标 模型 where the draft isn't much smaller.

生产 shops typically report 2-3× wall-clock speedup on chat, 3-5× on code 生成, and near-zero on creative writing.

```figure
speculative-decoding
```

## 动手构建

`code/main.py`:

- 一个参考 `speculative_decode(target, draft, prompt, K, temperature)` that implements the exact rejection rule and verifies it preserves the 目标's 分布 (empirical KL < 0.01 vs plain 目标 采样).
- 一个EAGLE-style tree drafter that builds a depth-K tree with top-p branching.
- 一个tree 注意力 掩码 builder that produces the right 因果 pattern for a verifier.
- 一个acceptance-rate harness that runs both on a tiny LM (distill one GPT-2-small from a GPT-2-medium 目标).

```python
def speculative_step(p_target, q_draft, K, temperature=1.0):
    """One round of speculative decoding. Returns list of accepted tokens."""
    # 1. Draft K tokens
    draft_tokens = []
    q_probs = []
    state = draft_state_init()
    for _ in range(K):
        probs = softmax(q_draft(state) / temperature)
        t = np.random.choice(len(probs), p=probs)
        draft_tokens.append(t)
        q_probs.append(probs[t])
        state = draft_step(state, t)

    # 2. Target computes p at every drafted position + 1 extra
    p_probs_all = target_forward_batched(p_target, draft_tokens, temperature)

    # 3. Accept/reject left-to-right
    accepted = []
    for k, tok in enumerate(draft_tokens):
        r = np.random.uniform()
        if r < p_probs_all[k][tok] / q_probs[k]:
            accepted.append(tok)
        else:
            residual = np.maximum(p_probs_all[k] - q_probs[k], 0)
            residual /= residual.sum()
            accepted.append(np.random.choice(len(residual), p=residual))
            return accepted
    # 4. All K accepted → sample bonus token from target
    accepted.append(np.random.choice(len(p_probs_all[-1]), p=p_probs_all[-1]))
    return accepted
```

## 实际使用

- **vLLM** and **SGLang** ship first-class 推测解码. Flags: `--speculative_model`, `--num_speculative_tokens`. EAGLE-2/3 support via the `--spec_decoding_algorithm eagle` flag.
- **NVIDIA TensorRT-LLM** supports Medusa and EAGLE trees natively.
- **参考 draft 模型**: `Qwen/Qwen3-0.6B-spec` (drafts for Qwen3-32B), `meta-llama/Llama-3.2-1B-Instruct-spec` (drafts for 70B).
- **Medusa 头** (Cai et al. 2024, "Medusa: Simple LLM 推理 Acceleration Framework with Multiple Decoding 头"): instead of a draft 模型, add K 并行 预测 头 to the 目标 itself. Simpler to deploy, slightly lower acceptance than EAGLE.

## 交付成果

这lesson produces `outputs/skill-speculative-tuning.md` — a skill that profiles a 目标 模型's workload and chooses: draft 模型, K (draft length), tree 宽度, temperature, and when to fall back to plain decode.

## 练习

1. Implement the exact rejection rule and empirically verify it. Run 10K 样本 via `speculative_decode` and via plain 目标 采样; 计算 TV distance between the two 输出 distributions. Should be < 0.01.

2. 计算 the speedup formula. Given fixed `α` and `K`, plot expected 词元 per target-forward. Find the optimal K for α ∈ {0.5, 0.7, 0.9}.

3. 训练 a tiny draft. Take a 124M GPT-2 目标 and distill a 30M GPT-2 draft on 100M 词元 with KL 损失. Measure `α` on held-out 文本. Expected: 0.6-0.7.

4. Implement EAGLE-style tree drafting. Instead of a 链, have the draft 输出 top-3 branches at each 深度. Build the tree 注意力 掩码. Verify the 目标 accepts the longest correct branch.

5. Measure failure modes. Run speculative decode at temperature=1.5 (high stochasticity). Show α collapses and the algorithm is slower than plain decode due to draft overhead.

## Key Terms

|Term|What people say|What it actually means|
|------|-----------------|------------------------|
|目标 模型|"The big 模型"|The slow, high-quality 模型 you want 样本 from (p 分布)|
|Draft 模型|"The speculator"|The small, fast predictor (q 分布); 5-30x smaller|
|K / draft length|"Look-ahead"|Number of speculated 词元 per verify pass|
|α / acceptance 速率|"Hit 速率"|Per-token 概率 that the draft's proposal is accepted|
|Exact rejection rule|"The accept test"|r < p/q compare that preserves 目标's 分布|
|Residual 分布|"Corrected p-q"|(p - q)+ /||(p - q)+||_1, the 分布 to 样本 from on rejection|
|Tree drafting|"Branching speculation"|Draft outputs a tree of candidates, verified in one pass with tree-structured 注意力 掩码|
|Tree 注意力 掩码|"Topological 掩码"|因果 掩码 encoding the tree 拓扑 so each 节点 attends only to its ancestors|
|Medusa 头|"并行 头"|K extra 预测 头 on the 目标 itself; no separate draft 模型|
|EAGLE 特征 reuse|"Hidden-state draft"|Draft 输入 is 目标's last 隐藏 状态, not raw 词元, shrinking the draft|
|Test-time simulation 损失|"EAGLE-3 训练"|训练 draft on outputs 匹配 目标's test-time 分布, not teacher forcing|

## 延伸阅读

- [Leviathan, Kalai, Matias, 2023 — "Fast Inference from Transformers via Speculative Decoding"](https://arxiv.org/abs/2211.17192) — the exact rejection rule and the theoretical speedup analysis
- [Chen, Borgeaud, Irving et al., 2023 — "Accelerating Large Language Model Decoding with Speculative Sampling"](https://arxiv.org/abs/2302.01318) — concurrent speculative-sampling paper at DeepMind
- [Cai, Li, Geng, Wang, Wang, Zhu, Dao, 2024 — "Medusa: Simple LLM Inference Acceleration Framework with Multiple Decoding Heads"](https://arxiv.org/abs/2401.10774) — parallel-heads 替代方案 to a draft 模型
- [Li, Wei, Zhang, Zhang, 2024 — "EAGLE: Speculative Sampling Requires Rethinking Feature Uncertainty"](https://arxiv.org/abs/2401.15077) — 特征 reuse and tree drafting
- [Li et al., 2024 — "EAGLE-2: Faster Inference of Language Models with Dynamic Draft Trees"](https://arxiv.org/abs/2406.16858) — dynamic tree 拓扑
- [Li et al., 2025 — "EAGLE-3: Scaling up Inference Acceleration of Large Language Models via Training-Time Test"](https://arxiv.org/abs/2503.01840) — train-time test-time 匹配
- [Fu, Haotian, Peng et al., 2024 — "Break the Sequential Dependency of LLM Inference Using Lookahead Decoding"](https://arxiv.org/abs/2402.02057) — Jacobi/lookahead decoding, a speculator-free 替代方案
