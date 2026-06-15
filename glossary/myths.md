# AI Myths Busted

AI 迷思破除

Common misconceptions about AI, ML, and deep learning. Each one explained with what's actually going on.
关于 AI、ML 和深度学习的常见误解。每一条都用真实情况说明。

---

## "AI understands language"
## “AI 理解语言”

**Reality:** LLMs predict the next token based on statistical patterns in training data. They have no understanding, no beliefs, no world model (that we can prove). They're very good at pattern matching across billions of examples. The output looks like understanding because the patterns are rich enough to cover most situations.
**现实：** LLM 基于训练数据中的统计模式预测下一个 token。它们没有理解、没有信念，也没有我们能证明存在的世界模型。它们非常擅长在数十亿样本中做模式匹配。输出看起来像理解，是因为这些模式足够丰富，覆盖了大多数情境。

**Why it matters:** If you treat an LLM as a reasoning engine, you'll be surprised when it confidently says wrong things. If you treat it as a pattern matcher, you'll design better systems around it.
**为什么重要：** 如果你把 LLM 当成推理引擎，它自信地说错话时你会很意外。如果你把它当成模式匹配器，你会围绕它设计出更好的系统。

---

## "More parameters = smarter model"
## “参数越多，模型越聪明”

**Reality:** A 7B parameter model trained on high-quality data with good techniques can outperform a 70B model trained on garbage. Chinchilla showed that most models were over-parameterized and under-trained. The quality and quantity of training data matters as much as model size. Phi-2 (2.7B) beat models 10x its size on many benchmarks.
**现实：** 一个用高质量数据和好技术训练的 7B 参数模型，可以超过一个用垃圾数据训练的 70B 模型。Chinchilla 表明，很多模型参数过多但训练不足。训练数据的质量和数量与模型大小同样重要。Phi-2，也就是 2.7B，在许多基准上超过了大它 10 倍的模型。

**Why it matters:** Don't default to the biggest model. Match model size to your task and budget.
**为什么重要：** 不要默认选择最大的模型。让模型大小匹配你的任务和预算。

---

## "Neural networks are black boxes"
## “神经网络是黑箱”

**Reality:** We have tools to understand what neural networks learn. Attention visualization shows what tokens the model focuses on. Probing classifiers reveal what information is stored in hidden representations. Mechanistic interpretability is finding actual circuits (induction heads, feature detectors). It's not complete transparency, but it's not a black box either.
**现实：** 我们有工具理解神经网络学到了什么。Attention 可视化能显示模型关注哪些 token。探针分类器能揭示隐藏表示中存储了什么信息。机制可解释性正在找到真实电路，比如 induction heads 和 feature detectors。它还不是完全透明，但也不是黑箱。

**Why it matters:** You can debug neural networks. Gradient analysis, activation visualization, and attention maps are real tools covered in this course.
**为什么重要：** 你可以调试神经网络。梯度分析、激活可视化和 attention map 都是真实工具，本课程会覆盖它们。

---

## "AI will replace programmers"
## “AI 会取代程序员”

**Reality:** AI changed programming, it didn't replace it. AI writes boilerplate. Humans design systems, make architectural decisions, review correctness, and handle the cases AI gets wrong. The role shifted from "write every line" to "review, direct, and architect." The best engineers use AI as a tool, not fear it as a replacement.
**现实：** AI 改变了编程，但没有取代编程。AI 可以写模板代码。人类设计系统、做架构决策、审查正确性，并处理 AI 出错的情况。角色从“写每一行代码”转向“审查、引导和设计架构”。最好的工程师把 AI 当工具，而不是害怕它取代自己。

**Why it matters:** You're learning AI engineering, which is programming + AI. Both skills together are more valuable than either alone.
**为什么重要：** 你正在学习 AI engineering，它是编程加 AI。两种技能结合起来，比单独任何一种都更有价值。

---

## "You need a PhD in math to do AI"
## “做 AI 需要数学博士”

**Reality:** You need high school math plus the specific topics in Phase 1 of this course. Linear algebra, calculus, probability, and optimization. You don't need proofs. You need intuition for what operations do and why they matter. If you can multiply matrices and take derivatives, you can build neural networks.
**现实：** 你需要高中数学，再加上本课程 Phase 1 中的特定主题。线性代数、微积分、概率和优化。你不需要证明。你需要的是直觉，知道这些运算在做什么、为什么重要。如果你能做矩阵乘法和求导，就能构建神经网络。

**Why it matters:** Phase 1 exists to give you exactly the math you need, nothing more.
**为什么重要：** Phase 1 的目的就是给你刚好需要的数学，不多也不少。

---

## "GPT stands for General Purpose Technology"
## “GPT 代表 General Purpose Technology”

**Reality:** GPT stands for Generative Pre-trained Transformer. Generative = it produces text. Pre-trained = trained once on a large corpus before being adapted. Transformer = the architecture from the 2017 "Attention Is All You Need" paper.
**现实：** GPT 代表 Generative Pre-trained Transformer。Generative 表示它生成文本。Pre-trained 表示先在大型语料上训练，再进行适配。Transformer 是 2017 年 “Attention Is All You Need” 论文中的架构。

---

## "Temperature makes the AI more creative"
## “Temperature 会让 AI 更有创造力”

**Reality:** Temperature scales the logits before softmax. Higher temperature = flatter probability distribution = more random token selection. Lower temperature = sharper distribution = more deterministic. It's not creativity, it's randomness. A high-temperature model doesn't think harder, it just considers less likely tokens.
**现实：** Temperature 会在 softmax 之前缩放 logits。更高的 temperature 意味着概率分布更平，token 选择更随机。更低的 temperature 意味着分布更尖，输出更确定。它不是创造力，而是随机性。高 temperature 模型不会思考得更努力，只是会考虑更低概率的 token。

**Why it matters:** When your output is too repetitive, raise temperature. When it's too chaotic, lower it. It's a randomness knob, nothing more.
**为什么重要：** 输出太重复时，提高 temperature。输出太混乱时，降低 temperature。它只是随机性旋钮，仅此而已。

---

## "Fine-tuning teaches the model new knowledge"
## “微调会教模型新知识”

**Reality:** Fine-tuning adjusts how the model uses existing knowledge, not what it knows. If information wasn't in the pre-training data, fine-tuning won't reliably add it. Fine-tuning is better for changing behavior (style, format, tone, task-specific patterns) than for adding facts. For new knowledge, use RAG.
**现实：** 微调调整的是模型如何使用已有知识，而不是模型知道什么。如果信息不在预训练数据中，微调不能可靠地添加它。微调更适合改变行为，比如风格、格式、语气和任务模式，而不是添加事实。需要新知识时，用 RAG。

**Why it matters:** If you need the model to know about your company's internal docs, use RAG. If you need it to respond in a specific format, fine-tune.
**为什么重要：** 如果你需要模型了解公司内部文档，用 RAG。如果你需要它按特定格式回答，用微调。

---

## "Bigger context window = better"
## “上下文窗口越大越好”

**Reality:** Models degrade on long contexts. The "lost in the middle" problem means models pay more attention to the beginning and end of long prompts and less to the middle. A 200K context window doesn't mean the model uses all 200K tokens equally well. Also, longer contexts cost more and are slower.
**现实：** 模型在长上下文中会退化。“lost in the middle” 问题意味着模型更关注长 prompt 的开头和结尾，而较少关注中间部分。200K 上下文窗口不代表模型能同样好地使用全部 200K token。另外，更长上下文更贵也更慢。

**Why it matters:** Don't dump everything into the context. Be selective. RAG with targeted retrieval beats stuffing the full document in.
**为什么重要：** 不要把所有东西都塞进上下文。要有选择。带有目标检索的 RAG 通常胜过塞入完整文档。

---

## "AI agents are autonomous"
## “AI agent 是自主的”

**Reality:** Current AI agents run in a loop: think, act, observe, repeat. They follow the pattern the harness defines. They don't have goals, plans, or self-awareness. They're reactive systems that use LLMs to decide what tool to call next. The "autonomy" comes from the loop, not from the AI.
**现实：** 当前 AI agent 在一个循环中运行，思考、行动、观察、重复。它们遵循 harness 定义的模式。它们没有目标、计划或自我意识。它们是反应式系统，用 LLM 决定下一步调用哪个工具。“自主性”来自循环，而不是来自 AI 本身。

**Why it matters:** When building agents, you're building the loop, the tools, and the guardrails. The LLM is just the decision-making component inside your system.
**为什么重要：** 构建 agent 时，你构建的是循环、工具和护栏。LLM 只是系统内部的决策组件。

---

## "Transformers understand order because of positional encoding"
## “Transformer 因为位置编码而理解顺序”

**Reality:** Transformers have no inherent sense of order. Self-attention treats input as a set, not a sequence. Positional encoding is a hack to inject order information by adding position-dependent vectors to the input. Different methods (sinusoidal, learned, RoPE, ALiBi) handle this differently. None of them truly give the model sequential understanding the way RNNs had it.
**现实：** Transformer 天生没有顺序感。Self-attention 把输入当成集合，而不是序列。位置编码是一种变通方式，通过向输入添加依赖位置的向量来注入顺序信息。不同方法，比如 sinusoidal、learned、RoPE、ALiBi，处理方式不同。没有哪一种真正让模型像 RNN 那样理解序列。

**Why it matters:** This is why positional encoding research is still active. It's a solved-enough problem for most uses, but it's fundamentally a workaround.
**为什么重要：** 这就是位置编码研究仍然活跃的原因。对多数用途来说它已经足够好，但从根本上说仍是一种绕法。

---

## "Pre-training is just reading the internet"
## “预训练就是读互联网”

**Reality:** Pre-training is next-token prediction on a massive corpus. The model learns to predict what comes next given what came before. Through this simple objective, it learns grammar, facts, reasoning patterns, code structure, and more. But it also learns internet nonsense, biases, and incorrect information. The data curation, filtering, and deduplication matter enormously.
**现实：** 预训练是在海量语料上做 next-token prediction。模型学习根据前文预测接下来会出现什么。通过这个简单目标，它学到语法、事实、推理模式、代码结构等。但它也会学到互联网垃圾、偏见和错误信息。数据筛选、过滤和去重非常重要。

**Why it matters:** Garbage in, garbage out. The quality of pre-training data is one of the biggest differentiators between models.
**为什么重要：** 垃圾进，垃圾出。预训练数据质量是模型之间最大的差异来源之一。

---

## "RLHF aligns AI with human values"
## “RLHF 让 AI 与人类价值观对齐”

**Reality:** RLHF aligns AI with the preferences of the specific humans who provided feedback. Those humans disagree with each other, have biases, and can't cover every situation. RLHF makes the model helpful and harmless in the ways the raters defined, not aligned with some universal human value system.
**现实：** RLHF 让 AI 对齐的是提供反馈的具体人群的偏好。这些人之间会有分歧，也会有偏见，并且不可能覆盖所有情况。RLHF 让模型在标注者定义的方式上更有帮助、更无害，而不是与某种普遍的人类价值系统对齐。

**Why it matters:** RLHF is a training technique, not a solution to alignment. It's one tool in a larger toolkit.
**为什么重要：** RLHF 是一种训练技术，不是 alignment 的完整解法。它只是更大工具箱中的一个工具。

---

## "Embeddings capture meaning"
## “Embedding 捕捉意义”

**Reality:** Embeddings capture statistical co-occurrence patterns. Words that appear in similar contexts get similar vectors. This correlates with meaning well enough to be useful, but it's not semantic understanding. "King - Man + Woman = Queen" works because of distributional patterns, not because the model understands monarchy or gender.
**现实：** Embedding 捕捉的是统计共现模式。出现在相似上下文中的词会得到相似向量。这与意义的相关性足够强，所以有用，但它不是语义理解。“King - Man + Woman = Queen” 能成立，是因为分布模式，而不是因为模型理解君主制或性别。

**Why it matters:** Embeddings are powerful for similarity search, clustering, and retrieval. But don't over-interpret what "similar" means.
**为什么重要：** Embedding 在相似度搜索、聚类和检索中很强大。但不要过度解读“相似”的含义。

---

## "Zero-shot means no training"
## “Zero-shot 表示没有训练”

**Reality:** Zero-shot means no task-specific examples at inference time. The model was still trained on billions of tokens. It just hasn't seen examples of this specific task format. It generalizes from pre-training patterns. Few-shot means giving a few examples in the prompt. Neither means the model learned without training.
**现实：** Zero-shot 表示推理时没有任务专用示例。模型仍然在数十亿 token 上训练过。它只是没有见过这个特定任务格式的示例。它从预训练模式中泛化。Few-shot 表示在 prompt 中给几个示例。两者都不表示模型没有训练就学会了。

---

## "AI models learn like humans"
## “AI 模型像人一样学习”

**Reality:** Humans learn from few examples, generalize across domains, and update beliefs continuously. Neural networks need millions of examples, generalize within their training distribution, and have fixed weights after training. The learning analogy is loose at best. Backpropagation is nothing like how biological neurons learn.
**现实：** 人类能从少量例子中学习，跨领域泛化，并持续更新信念。神经网络需要数百万样本，通常只在训练分布内泛化，训练后权重固定。学习这个类比最多只是宽泛类比。反向传播和生物神经元的学习方式完全不同。

**Why it matters:** Don't anthropomorphize models. It leads to wrong expectations about what they can and can't do.
**为什么重要：** 不要拟人化模型。这会让你对它们能做什么、不能做什么产生错误期待。

---

## "Scaling laws mean bigger is always better"
## “Scaling laws 表示越大越好”

**Reality:** Scaling laws describe predictable relationships between compute, data, and model size. They show diminishing returns: doubling parameters doesn't double performance. They also assume you scale data proportionally. Many practical improvements come from better architectures, training techniques, and data quality, not just scale.
**现实：** Scaling laws 描述 compute、data 和 model size 之间可预测的关系。它们显示收益递减，参数翻倍不会让性能翻倍。它们还假设数据也按比例扩大。很多实际提升来自更好的架构、训练技术和数据质量，而不只是规模。

**Why it matters:** A 7B model with good engineering can solve your problem. Don't reach for 70B by default.
**为什么重要：** 一个工程做得好的 7B 模型可能就能解决你的问题。不要默认上 70B。

---

## "Open source AI is the same as open weights"
## “开源 AI 等同于开放权重”

**Reality:** Most "open source" models are open weights. You get the model files but not the training data, training code, or data pipeline. True open source (like OLMo) releases everything: data, code, intermediate checkpoints, evaluation. Open weights is useful but not the same commitment as open source.
**现实：** 大多数“开源”模型其实是开放权重。你能拿到模型文件，但拿不到训练数据、训练代码或数据流水线。真正的开源，比如 OLMo，会发布所有东西，数据、代码、中间 checkpoint 和评估。开放权重很有用，但它和开源不是同一种承诺。

**Why it matters:** Know what you're getting. Open weights let you run and fine-tune. True open source lets you reproduce and understand.
**为什么重要：** 你要知道自己拿到的是什么。开放权重让你运行和微调。真正开源让你复现和理解。

---

## "Prompt engineering is not real engineering"
## “Prompt engineering 不是真工程”

**Reality:** Prompt engineering is system design. You're designing the interface between human intent and model behavior. Good prompt engineering requires understanding tokenization, attention patterns, context window limits, and output parsing. It's closer to API design than to "talking nicely to the AI."
**现实：** Prompt engineering 是系统设计。你设计的是人类意图和模型行为之间的接口。好的 prompt engineering 需要理解 tokenization、attention pattern、context window 限制和输出解析。它更接近 API 设计，而不是“好好跟 AI 说话”。

**Why it matters:** This course teaches prompt engineering as a real engineering discipline in Phase 11.
**为什么重要：** 本课程在 Phase 11 中把 prompt engineering 当成真实工程学科来教。

---

## "CNNs are outdated, everything is transformers now"
## “CNN 过时了，现在全是 Transformer”

**Reality:** Vision Transformers (ViT) beat CNNs on many benchmarks, but CNNs are still used extensively. They're faster for inference, work well on mobile/edge, need less data, and have useful inductive biases (translation invariance, local patterns). Many production vision systems still use CNNs. The best architectures often combine both.
**现实：** Vision Transformer，也就是 ViT，在很多基准上超过 CNN，但 CNN 仍被广泛使用。它们推理更快，适合移动端和边缘设备，需要更少数据，并且有有用的归纳偏置，比如平移不变性和局部模式。许多生产级视觉系统仍使用 CNN。最好的架构常常结合两者。

**Why it matters:** Learn both (Phases 4 and 7). Use what works for your constraints.
**为什么重要：** 两者都要学，也就是 Phase 4 和 Phase 7。根据你的约束选择有效方案。

---

## "You need massive compute to train useful models"
## “训练有用模型需要海量算力”

**Reality:** You need massive compute to pre-train foundation models. But fine-tuning, LoRA, and transfer learning let you adapt models on a single GPU. Many useful AI applications don't require training at all, just good prompting and RAG. The "compute barrier" is for building foundation models, not for using them.
**现实：** 预训练 foundation model 需要海量算力。但 fine-tuning、LoRA 和 transfer learning 可以让你在单张 GPU 上适配模型。很多有用的 AI 应用根本不需要训练，只需要好的 prompting 和 RAG。“算力门槛”主要在构建 foundation model，而不是使用它们。

**Why it matters:** You can build real AI applications with a laptop. This course proves it.
**为什么重要：** 你可以用笔记本构建真实 AI 应用。本课程会证明这一点。
