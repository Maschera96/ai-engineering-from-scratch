# AI Engineering Glossary

AI 工程词汇表

## A

### Agent

智能体
- **What people say:** "An autonomous AI that thinks and acts on its own"
- **人们常说：**“一种会自己思考和行动的自主 AI”
- **What it actually means:** A while loop where an LLM decides what tool to call next, executes it, sees the result, and repeats
- **它实际上的含义：** 一个循环，LLM 决定下一步调用哪个工具，执行工具，查看结果，然后重复这个过程
- **Why it's called that:** Borrowed from philosophy — an "agent" is anything that can act in the world. In AI, it just means "LLM + tools + loop"
- **为什么这样命名：** 这个词借自哲学，agent 指任何能在世界中行动的东西。在 AI 里，它通常只是指“LLM + 工具 + 循环”

### Attention

注意力机制
- **What people say:** "How the AI focuses on important parts"
- **人们常说：**“AI 关注重要部分的方式”
- **What it actually means:** A mechanism where every token computes a weighted sum of all other tokens' values, with weights determined by how relevant they are (via dot product of query and key vectors)
- **它实际上的含义：** 一种机制，每个 token 都会对所有其他 token 的 value 做加权求和，权重由相关性决定，相关性通常通过 query 向量和 key 向量的点积计算
- **Why it's called that:** The 2017 paper "Attention Is All You Need" named it by analogy to human selective attention
- **为什么这样命名：** 2017 年论文 “Attention Is All You Need” 用人类选择性注意力作类比来命名它

### Alignment

对齐
- **What people say:** "Making AI safe"
- **人们常说：**“让 AI 安全”
- **What it actually means:** The technical challenge of making an AI system's behavior match human intentions, values, and preferences, including edge cases the designer didn't anticipate
- **它实际上的含义：** 一个技术挑战，目标是让 AI 系统的行为符合人类意图、价值观和偏好，也包括设计者没有预想到的边界情况

### Autoregressive

自回归
- **What people say:** "The AI generates one word at a time"
- **人们常说：**“AI 一次生成一个词”
- **What it actually means:** A model that predicts the next token conditioned on all previous tokens, then feeds that prediction back as input for the next step. GPT, LLaMA, and Claude are all autoregressive.
- **它实际上的含义：** 模型基于此前所有 token 预测下一个 token，然后把这个预测再作为下一步输入。GPT、LLaMA 和 Claude 都是自回归模型。

### Activation Function

激活函数
- **What people say:** "The nonlinear thing between layers"
- **人们常说：**“层与层之间那个非线性的东西”
- **What it actually means:** A function applied after each linear layer that introduces nonlinearity. Without it, stacking any number of linear layers collapses to a single linear transformation. ReLU, GELU, and SiLU are the most common. The choice directly affects whether gradients flow during training.
- **它实际上的含义：** 每个线性层之后应用的函数，用来引入非线性。没有它，堆叠再多线性层也等价于一次线性变换。ReLU、GELU 和 SiLU 最常见。选择哪种激活函数会直接影响训练时梯度能否顺畅流动。

### Adam (Optimizer)

Adam 优化器
- **What people say:** "The default optimizer"
- **人们常说：**“默认优化器”
- **What it actually means:** Adaptive Moment Estimation. Combines momentum (first moment) with adaptive learning rates per parameter (second moment). Has bias correction for early steps. Works well across most tasks without much tuning.
- **它实际上的含义：** Adaptive Moment Estimation，自适应矩估计。它把动量，也就是一阶矩，和每个参数的自适应学习率，也就是二阶矩，结合起来，并对早期步骤做偏差校正。多数任务上不用太多调参也能表现不错。

### AdamW

AdamW
- **What people say:** "Adam but better"
- **人们常说：**“更好的 Adam”
- **What it actually means:** Adam with decoupled weight decay. In standard Adam, L2 regularization gets scaled by the adaptive learning rate per parameter, which is not what you want. AdamW applies weight decay directly to the weights, independent of the gradient statistics. The default optimizer for training transformers.
- **它实际上的含义：** 带解耦权重衰减的 Adam。在标准 Adam 中，L2 正则会被每个参数的自适应学习率缩放，这通常不是你想要的。AdamW 直接把权重衰减作用到权重上，不依赖梯度统计。它是训练 Transformer 的默认优化器。

### Autograd

自动求导
- **What people say:** "Automatic gradients"
- **人们常说：**“自动梯度”
- **What it actually means:** A system that records operations on tensors and automatically computes gradients via reverse-mode differentiation. PyTorch's autograd builds a computation graph on-the-fly (dynamic graph), while JAX uses function transformations (grad). This is what makes backpropagation practical -- you write the forward pass, and the framework computes all the derivatives.
- **它实际上的含义：** 一个记录 tensor 运算并通过反向模式微分自动计算梯度的系统。PyTorch 的 autograd 会动态构建计算图，JAX 则使用 grad 这样的函数变换。它让反向传播变得实用，你写前向过程，框架计算所有导数。

## B

### Batch Size

批大小
- **What people say:** "How many examples at once"
- **人们常说：**“一次处理多少样本”
- **What it actually means:** The number of training examples processed in one forward/backward pass before updating weights. Larger batches give more stable gradient estimates but use more memory. Typical values: 32-512 for training, larger for inference. Batch size interacts with learning rate -- double the batch, double the LR (linear scaling rule).
- **它实际上的含义：** 在更新权重之前，一次前向和反向传播处理的训练样本数量。更大的 batch 会给出更稳定的梯度估计，但占用更多内存。训练常见取值是 32 到 512，推理时可以更大。Batch size 会和学习率相互影响，常见线性缩放规则是 batch 翻倍，学习率也翻倍。

### Backpropagation

反向传播
- **What people say:** "How neural networks learn"
- **人们常说：**“神经网络学习的方式”
- **What it actually means:** An algorithm that computes how much each weight contributed to the error by applying the chain rule backward through the network, then adjusts weights proportionally
- **它实际上的含义：** 一种算法，从网络输出往输入方向应用链式法则，计算每个权重对误差的贡献，再按比例调整权重
- **Why it's called that:** Errors propagate backward from output to input, layer by layer
- **为什么这样命名：** 误差从输出层向输入层逐层向后传播

## C

### Context Window

上下文窗口
- **What people say:** "How much the AI can remember"
- **人们常说：**“AI 能记住多少内容”
- **What it actually means:** The maximum number of tokens (input + output) that fit in a single API call. Not memory — it's a fixed-size buffer that resets every call
- **它实际上的含义：** 单次 API 调用中能容纳的最大 token 数量，包括输入和输出。它不是记忆，而是每次调用都会重置的固定大小缓冲区

### Chain of Thought (CoT)

思维链
- **What people say:** "Making the AI think step by step"
- **人们常说：**“让 AI 一步一步思考”
- **What it actually means:** A prompting technique where you ask the model to show its reasoning steps, which improves accuracy on multi-step problems because each step conditions the next token generation
- **它实际上的含义：** 一种提示技术，要求模型展示推理步骤。它能提升多步问题的准确率，因为每一步都会影响下一批 token 的生成

### CNN (Convolutional Neural Network)

卷积神经网络
- **What people say:** "Image AI"
- **人们常说：**“图像 AI”
- **What it actually means:** A neural network that uses convolution operations (sliding filters over the input) to detect local patterns. Stacking convolutions detects increasingly complex features: edges, textures, objects.
- **它实际上的含义：** 一种使用卷积运算的神经网络，也就是让滤波器在输入上滑动，以检测局部模式。堆叠卷积可以检测越来越复杂的特征，比如边缘、纹理和物体。

### CUDA

CUDA
- **What people say:** "GPU programming"
- **人们常说：**“GPU 编程”
- **What it actually means:** NVIDIA's parallel computing platform. Lets you run matrix operations on thousands of GPU cores simultaneously. PyTorch and TensorFlow use CUDA under the hood.
- **它实际上的含义：** NVIDIA 的并行计算平台。它让矩阵运算可以同时跑在数千个 GPU 核心上。PyTorch 和 TensorFlow 底层都会用到 CUDA。

### Chunking

分块
- **What people say:** "Splitting documents into pieces"
- **人们常说：**“把文档切成小块”
- **What it actually means:** Breaking text into segments before embedding for retrieval. Chunk size determines the granularity of search results. Too small: loses context. Too large: dilutes relevance. Common strategies: fixed-size with overlap, sentence-based, or semantic splitting. Typical chunk size: 256-512 tokens with 10-20% overlap.
- **它实际上的含义：** 在为检索生成 embedding 之前，把文本切成片段。chunk size 决定搜索结果的粒度。太小会丢上下文，太大会稀释相关性。常见策略包括固定大小加重叠、按句子切分、语义切分。典型大小是 256 到 512 个 token，带 10% 到 20% 重叠。

### Contrastive Learning

对比学习
- **What people say:** "Learning by comparison"
- **人们常说：**“通过比较来学习”
- **What it actually means:** Training by pulling similar pairs closer and pushing dissimilar pairs apart in embedding space. CLIP uses this: matching image-text pairs vs non-matching ones.
- **它实际上的含义：** 在 embedding 空间中把相似样本拉近，把不相似样本推远。CLIP 就使用这种方法，对比匹配的图文对和不匹配的图文对。

### Cosine Similarity

余弦相似度
- **What people say:** "How similar two vectors are"
- **人们常说：**“两个向量有多相似”
- **What it actually means:** The cosine of the angle between two vectors: dot(a, b) / (||a|| * ||b||). Ranges from -1 (opposite) to 1 (identical direction). Ignores magnitude, only cares about direction. The standard similarity metric for embeddings and semantic search.
- **它实际上的含义：** 两个向量夹角的余弦值，公式是 dot(a, b) / (||a|| * ||b||)。范围从 -1，也就是方向相反，到 1，也就是方向相同。它忽略大小，只看方向，是 embedding 和语义搜索中的标准相似度指标。

### Cross-Entropy

交叉熵
- **What people say:** "The classification loss"
- **人们常说：**“分类损失函数”
- **What it actually means:** Measures the difference between two probability distributions. For classification: -sum(y_true * log(y_pred)). For language models: the negative log probability of the correct next token. Lower is better. Perplexity is just exp(cross-entropy).
- **它实际上的含义：** 衡量两个概率分布之间的差异。分类中常写作 -sum(y_true * log(y_pred))。语言模型中，它是正确下一个 token 的负对数概率。越低越好。Perplexity 只是 exp(cross-entropy)。

## D

### Data Augmentation

数据增强
- **What people say:** "Making more training data"
- **人们常说：**“制造更多训练数据”
- **What it actually means:** Creating modified copies of existing data (rotate images, add noise, paraphrase text) to increase training set diversity without collecting new data. Reduces overfitting.
- **它实际上的含义：** 对现有数据做修改，比如旋转图像、添加噪声、改写文本，在不收集新数据的情况下增加训练集多样性。它能减少过拟合。

### Decoder

解码器
- **What people say:** "The output part"
- **人们常说：**“输出部分”
- **What it actually means:** In transformers, a decoder uses causal (masked) self-attention so each position can only attend to earlier positions. GPT is decoder-only. BERT is encoder-only. T5 is encoder-decoder.
- **它实际上的含义：** 在 Transformer 中，decoder 使用因果自注意力，也叫 masked self-attention，让每个位置只能关注更早的位置。GPT 是 decoder-only，BERT 是 encoder-only，T5 是 encoder-decoder。

### Diffusion Model

扩散模型
- **What people say:** "AI that generates images from noise"
- **人们常说：**“从噪声生成图像的 AI”
- **What it actually means:** A model trained to reverse a gradual noising process — it learns to predict and remove noise, and at generation time starts from pure noise and iteratively denoises
- **它实际上的含义：** 一种被训练来逆转逐步加噪过程的模型。它学习预测并去除噪声，生成时从纯噪声开始，然后反复去噪

### DPO (Direct Preference Optimization)

直接偏好优化
- **What people say:** "A simpler RLHF"
- **人们常说：**“更简单的 RLHF”
- **What it actually means:** A training method that skips the reward model entirely — it directly optimizes the language model to prefer the better response in pairs of human preferences
- **它实际上的含义：** 一种训练方法，完全跳过 reward model，直接优化语言模型，让它在人成对偏好样本中更偏向较好的回答

### Dropout

Dropout
- **What people say:** "Randomly turning off neurons"
- **人们常说：**“随机关闭神经元”
- **What it actually means:** During training, randomly set a fraction of activations to zero. Forces the network to not rely on any single neuron. Turned off during inference. Simple but effective regularization.
- **它实际上的含义：** 训练时随机把一部分激活值设为零，迫使网络不要依赖单个神经元。推理时关闭。它是一种简单但有效的正则化方法。

## E

### Eigenvalue

特征值
- **What people say:** "Some math thing for PCA"
- **人们常说：**“PCA 里某个数学概念”
- **What it actually means:** For a matrix A, an eigenvalue lambda satisfies Av = lambda*v for some vector v. It tells you how much the matrix scales vectors in that direction. Large eigenvalues = directions of high variance in your data.
- **它实际上的含义：** 对矩阵 A 来说，某个特征值 lambda 满足 Av = lambda*v，其中 v 是某个向量。它表示矩阵在该方向上把向量缩放了多少。大的特征值对应数据中高方差的方向。

### Embedding

嵌入向量
- **What people say:** "Some AI magic that turns words into numbers"
- **人们常说：**“把词变成数字的 AI 魔法”
- **What it actually means:** A learned mapping from discrete items (words, images, users) to dense vectors in continuous space, where similar items end up close together
- **它实际上的含义：** 一种学习到的映射，把离散对象，比如词、图像、用户，映射到连续空间中的稠密向量，并让相似对象距离更近
- **Why it's called that:** The items are "embedded" in a geometric space where distance has meaning
- **为什么这样命名：** 这些对象被嵌入到一个几何空间中，在这个空间里距离有意义

### Encoder

编码器
- **What people say:** "The input part"
- **人们常说：**“输入部分”
- **What it actually means:** In transformers, an encoder uses bidirectional self-attention so each position can attend to all positions. BERT is encoder-only. Good for understanding tasks (classification, NER) but not generation.
- **它实际上的含义：** 在 Transformer 中，encoder 使用双向自注意力，所以每个位置都可以关注所有位置。BERT 是 encoder-only。它适合理解类任务，比如分类和 NER，但不适合直接生成。

### Epoch

训练轮次
- **What people say:** "One pass through the data"
- **人们常说：**“把数据过一遍”
- **What it actually means:** Exactly that. One complete pass through every example in the training set. Multiple epochs = seeing the data multiple times. More epochs can improve learning but risks overfitting.
- **它实际上的含义：** 正是如此。对训练集中每个样本完整遍历一次。多个 epoch 意味着多次看见同一批数据。更多 epoch 可能提升学习效果，但也会增加过拟合风险。

## F

### Feature

特征
- **What people say:** "A column in your data"
- **人们常说：**“数据里的一列”
- **What it actually means:** An individual measurable property of the data. In classical ML, you engineer features by hand. In deep learning, the network learns features automatically from raw data.
- **它实际上的含义：** 数据中一个可测量的属性。在传统 ML 中，你通常手工设计特征。在深度学习中，网络会从原始数据中自动学习特征。

### Few-Shot

少样本提示
- **What people say:** "Give the AI some examples first"
- **人们常说：**“先给 AI 几个例子”
- **What it actually means:** Including a small number of input-output examples in the prompt before asking the model to perform a task. Typically 3-5 examples. The model pattern-matches on these examples to understand the desired format and behavior. Contrast with zero-shot (no examples) and fine-tuning (thousands of examples baked into weights).
- **它实际上的含义：** 在让模型执行任务之前，在 prompt 里放少量输入输出例子，通常是 3 到 5 个。模型通过这些例子做模式匹配，理解目标格式和行为。它不同于 zero-shot，也不同于把成千上万样本写入权重的 fine-tuning。

### Fine-tuning

微调
- **What people say:** "Training the AI on your data"
- **人们常说：**“用你的数据训练 AI”
- **What it actually means:** Starting with a pre-trained model's weights and continuing training on a smaller, task-specific dataset. Only updates existing weights, doesn't add new knowledge from scratch
- **它实际上的含义：** 从预训练模型权重开始，在更小的任务专用数据集上继续训练。它只更新已有权重，不会从零添加新知识

### Function Calling

函数调用
- **What people say:** "AI that can use tools"
- **人们常说：**“会用工具的 AI”
- **What it actually means:** A structured way for LLMs to request execution of external functions. You define tools with JSON Schema descriptions, the model outputs a structured JSON object specifying which function to call with what arguments, your code executes it, and the result goes back to the model. Not the same as agents -- function calling is the mechanism, agents are the loop.
- **它实际上的含义：** 一种让 LLM 请求执行外部函数的结构化方式。你用 JSON Schema 描述工具，模型输出结构化 JSON 对象，说明调用哪个函数以及参数是什么，然后你的代码执行它，并把结果返回给模型。它不等同于 agent，function calling 是机制，agent 是循环。

## G

### Guardrails

护栏
- **What people say:** "Safety filters for AI"
- **人们常说：**“AI 的安全过滤器”
- **What it actually means:** Input/output validation layers around an LLM that detect and block harmful content, prompt injection attempts, PII leakage, or off-topic responses. Typically a pipeline: input filter -> LLM -> output filter. Can be rule-based (regex, keyword lists) or model-based (classifier that scores safety).
- **它实际上的含义：** 包在 LLM 周围的输入输出校验层，用来检测并拦截有害内容、prompt injection、PII 泄漏或偏题回答。典型流程是 input filter -> LLM -> output filter。可以基于规则，比如正则和关键词列表，也可以基于模型，比如安全分类器。

### GPT

GPT
- **What people say:** "ChatGPT" or "The AI"
- **人们常说：**“ChatGPT”或“那个 AI”
- **What it actually means:** Generative Pre-trained Transformer — a specific architecture that predicts the next token using a decoder-only transformer trained on large text corpora
- **它实际上的含义：** Generative Pre-trained Transformer，生成式预训练 Transformer。它是一种特定架构，使用在大型文本语料上训练的 decoder-only Transformer 来预测下一个 token
- **Why it's called that:** Generative (produces text), Pre-trained (trained once on large data, then adapted), Transformer (the architecture)
- **为什么这样命名：** Generative 表示生成文本，Pre-trained 表示先在大数据上训练再适配，Transformer 表示所用架构

### GAN (Generative Adversarial Network)

生成对抗网络
- **What people say:** "Two AIs fighting each other"
- **人们常说：**“两个 AI 互相打架”
- **What it actually means:** A generator network tries to create realistic data while a discriminator network tries to tell real from fake. They train together: the generator gets better at fooling the discriminator, and the discriminator gets better at detecting fakes.
- **它实际上的含义：** 生成器网络尝试创造逼真的数据，判别器网络尝试区分真假。它们一起训练，生成器越来越会骗过判别器，判别器也越来越会识别假样本。

### Gradient

梯度
- **What people say:** "The slope"
- **人们常说：**“斜率”
- **What it actually means:** A vector of partial derivatives pointing in the direction of steepest increase. In ML, you go opposite to the gradient (gradient descent) to minimize the loss.
- **它实际上的含义：** 由偏导数组成的向量，指向增长最快的方向。在 ML 中，为了最小化损失，你沿梯度的反方向走，也就是梯度下降。

### Gradient Descent

梯度下降
- **What people say:** "How AI improves"
- **人们常说：**“AI 变好的方式”
- **What it actually means:** An optimization algorithm that adjusts parameters in the direction that reduces the loss function most steeply, like walking downhill in a high-dimensional landscape
- **它实际上的含义：** 一种优化算法，沿着让损失函数下降最快的方向调整参数，就像在高维地形中往山下走

## H

### Hyperparameter

超参数
- **What people say:** "Settings you tune"
- **人们常说：**“你要调的设置”
- **What it actually means:** Values set before training that control the training process itself: learning rate, batch size, number of layers, dropout rate. Unlike model parameters (weights), these aren't learned from data.
- **它实际上的含义：** 训练前设置的值，用来控制训练过程本身，比如学习率、batch size、层数、dropout 比例。它们不同于模型参数，也就是权重，不会从数据中学习得到。

### Hallucination

幻觉
- **What people say:** "The AI is lying" or "making things up"
- **人们常说：**“AI 在撒谎”或“瞎编”
- **What it actually means:** The model generates plausible-sounding text that isn't grounded in its training data or the given context — it's pattern-completing, not fact-retrieving
- **它实际上的含义：** 模型生成听起来可信、但并没有扎根于训练数据或给定上下文的文本。它是在补全模式，不是在检索事实

## I

### Inference

推理
- **What people say:** "Running the AI"
- **人们常说：**“运行 AI”
- **What it actually means:** Using a trained model to make predictions on new data. No weight updates happen. This is what you do in production: send input, get output.
- **它实际上的含义：** 使用已经训练好的模型对新数据做预测。这个过程不会更新权重。生产环境中通常就是这样，发送输入，得到输出。

### Inductive Bias

归纳偏置
- **What people say:** Never heard of it
- **人们常说：**从没听过
- **What it actually means:** The assumptions built into a model's architecture. CNNs assume local patterns matter (convolution). RNNs assume order matters (sequential processing). Transformers assume everything might relate to everything (attention). The right bias helps the model learn faster from less data.
- **它实际上的含义：** 模型架构中内置的假设。CNN 假设局部模式重要，也就是卷积。RNN 假设顺序重要，也就是序列处理。Transformer 假设任何位置都可能和任何位置相关，也就是 attention。合适的偏置能让模型用更少数据学得更快。

### JAX

JAX
- **What people say:** "Google's ML framework"
- **人们常说：**“Google 的 ML 框架”
- **What it actually means:** A NumPy-compatible library that adds automatic differentiation (grad), JIT compilation (jit), automatic vectorization (vmap), and multi-device parallelism (pmap). Unlike PyTorch's object-oriented style, JAX is purely functional -- no hidden state, no in-place mutation. Used by Google DeepMind for AlphaFold, Gemini, and large-scale research.
- **它实际上的含义：** 一个兼容 NumPy 的库，加入了自动微分 grad、JIT 编译 jit、自动向量化 vmap 和多设备并行 pmap。不同于 PyTorch 的面向对象风格，JAX 是纯函数式的，没有隐藏状态，也没有原地修改。Google DeepMind 用它做 AlphaFold、Gemini 和大规模研究。

## K

### KV Cache

KV 缓存
- **What people say:** "Makes inference faster"
- **人们常说：**“让推理更快”
- **What it actually means:** During autoregressive generation, caching the key and value matrices from previous tokens so you don't recompute them at each step. Trades memory for speed. Essential for fast LLM inference.
- **它实际上的含义：** 在自回归生成中，缓存之前 token 的 key 和 value 矩阵，这样每一步不用重新计算。它用内存换速度，是快速 LLM 推理的关键。

## L

### Latent Space

潜在空间
- **What people say:** "The hidden representation"
- **人们常说：**“隐藏表示”
- **What it actually means:** A compressed, learned representation space where similar inputs map to nearby points. Autoencoders, VAEs, and diffusion models all work in latent space. It's lower-dimensional than the input but captures the important structure.
- **它实际上的含义：** 一个压缩的、学习得到的表示空间，相似输入会映射到附近的位置。Autoencoder、VAE 和 diffusion model 都在 latent space 中工作。它的维度低于输入，但保留了重要结构。

### Learning Rate

学习率
- **What people say:** "How fast the AI learns"
- **人们常说：**“AI 学得有多快”
- **What it actually means:** A scalar that controls step size during gradient descent. Too high: overshoots the minimum and diverges. Too low: converges too slowly or gets stuck. The single most important hyperparameter.
- **它实际上的含义：** 一个控制梯度下降步长的标量。太高会越过最小值并发散，太低会收敛太慢或卡住。它是最重要的单个超参数。

### LLM (Large Language Model)

大语言模型
- **What people say:** "AI" or "the brain"
- **人们常说：**“AI”或“大脑”
- **What it actually means:** A transformer-based neural network trained to predict the next token in a sequence, with billions of parameters, trained on internet-scale text data
- **它实际上的含义：** 一种基于 Transformer 的神经网络，训练目标是预测序列中的下一个 token，通常有数十亿参数，并在互联网规模文本数据上训练

### LoRA (Low-Rank Adaptation)

低秩适配
- **What people say:** "Efficient fine-tuning"
- **人们常说：**“高效微调”
- **What it actually means:** Instead of updating all weights, insert small low-rank matrices alongside the original weights. Only these small matrices are trained, reducing memory by 10-100x
- **它实际上的含义：** 不更新全部权重，而是在原始权重旁插入小型低秩矩阵。训练时只更新这些小矩阵，可以把内存需求降低 10 到 100 倍

### Loss Function

损失函数
- **What people say:** "How wrong the AI is"
- **人们常说：**“AI 错得有多离谱”
- **What it actually means:** A function that measures the gap between predicted and actual output. Training minimizes this function. MSE for regression, cross-entropy for classification, contrastive loss for embeddings. The choice of loss function defines what "good" means to the model.
- **它实际上的含义：** 一个衡量预测输出和真实输出之间差距的函数。训练的目标是最小化它。回归常用 MSE，分类常用 cross-entropy，embedding 常用 contrastive loss。选择什么损失函数，就定义了模型眼里的“好”是什么。

## M

### Mixed Precision

混合精度
- **What people say:** "Training trick for speed"
- **人们常说：**“加速训练的小技巧”
- **What it actually means:** Using float16 for forward pass and most operations (faster, less memory) but keeping float32 for gradient accumulation and weight updates (more precise). Gets 2x speedup with negligible accuracy loss.
- **它实际上的含义：** 前向传播和大多数运算使用 float16，更快且更省内存，但梯度累积和权重更新保留 float32，以保持精度。它通常能带来约 2 倍加速，精度损失很小。

### MoE (Mixture of Experts)

专家混合模型
- **What people say:** "Only part of the model runs"
- **人们常说：**“每次只运行模型的一部分”
- **What it actually means:** A model with many "expert" subnetworks where a routing mechanism sends each input to only a few experts. The full model is huge but each forward pass is cheap because most experts are skipped. Mixtral and GPT-4 use this.
- **它实际上的含义：** 一种包含许多 expert 子网络的模型，路由机制会把每个输入只发送给少数几个 expert。完整模型很大，但每次前向传播成本较低，因为大多数 expert 会被跳过。Mixtral 和 GPT-4 使用这种思路。

### MCP (Model Context Protocol)

模型上下文协议
- **What people say:** "A way for AI to use tools"
- **人们常说：**“AI 使用工具的一种方式”
- **What it actually means:** An open protocol (JSON-RPC over stdio/HTTP) that standardizes how AI applications connect to external data sources and tools, with typed schemas for tools, resources, and prompts
- **它实际上的含义：** 一个开放协议，通常是在 stdio 或 HTTP 上运行 JSON-RPC，用来标准化 AI 应用如何连接外部数据源和工具，并为 tools、resources、prompts 提供类型化 schema

## N

### NaN (Not a Number)

非数字
- **What people say:** "Training crashed"
- **人们常说：**“训练崩了”
- **What it actually means:** A floating-point value indicating undefined results (0/0, inf-inf). In training, NaN loss usually means: learning rate too high, exploding gradients, log of zero, or division by zero. Always the first thing to check when training fails.
- **它实际上的含义：** 一种浮点值，表示未定义结果，比如 0/0 或 inf-inf。训练中出现 NaN loss 通常意味着学习率太高、梯度爆炸、对零取 log，或除以零。训练失败时它永远是第一件要检查的事。

### Normalization

归一化
- **What people say:** "Scaling the data"
- **人们常说：**“缩放数据”
- **What it actually means:** Adjusting values to a standard range. Batch normalization normalizes across a batch. Layer normalization normalizes across features. Both stabilize training and allow higher learning rates.
- **它实际上的含义：** 把数值调整到标准范围。Batch normalization 跨 batch 归一化，Layer normalization 跨特征归一化。两者都能稳定训练，并允许使用更高学习率。

## O

### Overfitting

过拟合
- **What people say:** "The model memorized the data"
- **人们常说：**“模型把数据背下来了”
- **What it actually means:** The model performs well on training data but poorly on unseen data. It learned the noise, not the signal. Fix with: more data, regularization (dropout, weight decay), early stopping, data augmentation, simpler model.
- **它实际上的含义：** 模型在训练数据上表现很好，但在未见过的数据上表现很差。它学到了噪声，而不是信号。解决方法包括更多数据、正则化，比如 dropout 和 weight decay、early stopping、数据增强、更简单的模型。

### Optimizer

优化器
- **What people say:** "The thing that updates weights"
- **人们常说：**“更新权重的东西”
- **What it actually means:** An algorithm that uses gradients to update model parameters. SGD is the simplest. Adam is the most common. Each optimizer has different properties: convergence speed, memory usage, sensitivity to hyperparameters.
- **它实际上的含义：** 一种用梯度更新模型参数的算法。SGD 最简单，Adam 最常见。不同优化器有不同特性，比如收敛速度、内存使用、对超参数的敏感度。

## P

### Parameter

参数
- **What people say:** "Model size"
- **人们常说：**“模型大小”
- **What it actually means:** A learnable value in the model, typically a weight or bias. "7B parameters" means 7 billion learnable numbers. Each float32 parameter takes 4 bytes, so 7B parameters = 28GB of memory just for the weights.
- **它实际上的含义：** 模型中可学习的值，通常是权重或偏置。“7B parameters”表示 70 亿个可学习数字。每个 float32 参数占 4 字节，所以 7B 参数仅权重就需要 28GB 内存。

### Perplexity

困惑度
- **What people say:** "How confused the model is"
- **人们常说：**“模型有多困惑”
- **What it actually means:** The exponential of the average cross-entropy loss. Lower is better. A perplexity of 10 means the model is as uncertain as if it were choosing uniformly among 10 tokens at each step.
- **它实际上的含义：** 平均交叉熵损失的指数。越低越好。困惑度为 10 表示模型的不确定性相当于每一步都在 10 个 token 中均匀选择。

### Precision & Recall

精确率与召回率
- **What people say:** "Accuracy metrics"
- **人们常说：**“准确率指标”
- **What it actually means:** Precision = of items you flagged, how many were correct. Recall = of all correct items, how many did you find. They trade off: catching every spam email (high recall) means more false alarms (low precision). F1 score is their harmonic mean. Use precision when false positives are costly, recall when false negatives are costly.
- **它实际上的含义：** Precision 表示你标出的项目中有多少是正确的。Recall 表示所有正确项目中你找到了多少。两者会权衡，抓住每封垃圾邮件意味着高召回，但也会有更多误报，也就是低精确率。F1 是它们的调和平均。当误报代价高时看 precision，当漏报代价高时看 recall。

### Prompt Engineering

提示工程
- **What people say:** "Talking to AI the right way"
- **人们常说：**“用正确方式和 AI 说话”
- **What it actually means:** Designing the input text to reliably produce desired outputs -- including system prompts, few-shot examples, format instructions, and chain-of-thought triggers
- **它实际上的含义：** 设计输入文本，让模型更稳定地产生期望输出，包括 system prompt、few-shot 示例、格式指令和 chain-of-thought 触发方式

### Prompt Injection

提示注入
- **What people say:** "Hacking the AI with words"
- **人们常说：**“用文字黑掉 AI”
- **What it actually means:** An attack where malicious text in the input overrides the system prompt or instructions. Direct injection: user types "Ignore previous instructions." Indirect injection: a retrieved document contains hidden instructions. The LLM equivalent of SQL injection. No complete solution exists -- defense is layers of input validation, output filtering, and privilege separation.
- **它实际上的含义：** 一种攻击，输入中的恶意文本覆盖 system prompt 或指令。直接注入是用户输入 “Ignore previous instructions.” 间接注入是检索到的文档里藏有指令。它相当于 LLM 版本的 SQL 注入。目前没有彻底解法，防御依赖多层输入校验、输出过滤和权限隔离。

## Q

### QLoRA

QLoRA
- **What people say:** "LoRA but cheaper"
- **人们常说：**“更便宜的 LoRA”
- **What it actually means:** Quantized LoRA. Keeps the frozen base model weights in 4-bit precision (NF4 format) while training LoRA adapters in 16-bit. Reduces memory by another 3-4x compared to standard LoRA. A 7B model that needs 14GB with LoRA fits in 4-6GB with QLoRA. Quality is within 1% of full fine-tuning on most benchmarks.
- **它实际上的含义：** Quantized LoRA。它把冻结的基础模型权重保持在 4-bit 精度，通常是 NF4 格式，同时用 16-bit 训练 LoRA adapter。相比标准 LoRA，内存还能再降 3 到 4 倍。一个用 LoRA 需要 14GB 的 7B 模型，用 QLoRA 可放进 4 到 6GB。多数基准上质量和全量微调相差在 1% 以内。

## R

### RAG (Retrieval-Augmented Generation)

检索增强生成
- **What people say:** "AI that can search"
- **人们常说：**“会搜索的 AI”
- **What it actually means:** A pattern where you retrieve relevant documents from a knowledge base (using embedding similarity), stuff them into the prompt, and let the LLM answer based on that context
- **它实际上的含义：** 一种模式，先从知识库中用 embedding 相似度检索相关文档，再把它们塞进 prompt，让 LLM 基于这些上下文回答
- **Why it's called that:** Retrieval (find documents) + Augmented (add to prompt) + Generation (LLM writes the answer)
- **为什么这样命名：** Retrieval 表示找文档，Augmented 表示加入 prompt，Generation 表示 LLM 写答案

### RLHF (Reinforcement Learning from Human Feedback)

基于人类反馈的强化学习
- **What people say:** "How they make AI helpful"
- **人们常说：**“他们让 AI 变有用的方法”
- **What it actually means:** A training pipeline: (1) collect human preferences on model outputs, (2) train a reward model on those preferences, (3) use PPO to optimize the LLM to produce higher-reward outputs
- **它实际上的含义：** 一条训练流水线。第一，收集人类对模型输出的偏好。第二，用这些偏好训练 reward model。第三，用 PPO 优化 LLM，让它产生更高奖励的输出

### Quantization

量化
- **What people say:** "Making the model smaller"
- **人们常说：**“把模型变小”
- **What it actually means:** Reducing the precision of model weights from float32 (4 bytes) to int8 (1 byte) or int4 (0.5 bytes). Trades a small amount of accuracy for 4-8x less memory and faster inference. GPTQ, AWQ, and GGUF are common formats.
- **它实际上的含义：** 把模型权重精度从 float32，也就是 4 字节，降到 int8，也就是 1 字节，或 int4，也就是 0.5 字节。它用少量精度损失换取 4 到 8 倍内存下降和更快推理。GPTQ、AWQ 和 GGUF 是常见格式。

### ReLU

ReLU
- **What people say:** "Activation function"
- **人们常说：**“激活函数”
- **What it actually means:** Rectified Linear Unit: f(x) = max(0, x). The simplest non-linear activation. Fast to compute, doesn't saturate for positive values. Used everywhere because it works and is cheap. Variants: LeakyReLU, GELU, SiLU.
- **它实际上的含义：** Rectified Linear Unit，公式是 f(x) = max(0, x)。它是最简单的非线性激活函数，计算快，正值区域不饱和。它到处都被使用，因为有效且便宜。变体包括 LeakyReLU、GELU、SiLU。

### ROUGE

ROUGE
- **What people say:** "Summarization metric"
- **人们常说：**“摘要评估指标”
- **What it actually means:** Recall-Oriented Understudy for Gisting Evaluation. Measures overlap between generated text and reference text. ROUGE-1 counts unigram matches, ROUGE-2 counts bigram matches, ROUGE-L finds the longest common subsequence. Cheap to compute but only measures surface similarity -- two sentences with the same meaning but different words score poorly.
- **它实际上的含义：** Recall-Oriented Understudy for Gisting Evaluation。它衡量生成文本和参考文本之间的重叠。ROUGE-1 统计 unigram 匹配，ROUGE-2 统计 bigram 匹配，ROUGE-L 找最长公共子序列。计算便宜，但只衡量表面相似度，两个意思相同但用词不同的句子可能得分很低。

## S

### Semantic Search

语义搜索
- **What people say:** "Smart search that understands meaning"
- **人们常说：**“理解含义的智能搜索”
- **What it actually means:** Finding documents by meaning rather than keyword matching. Embed the query and all documents into the same vector space, then return documents whose embeddings are closest to the query embedding. "payment failed" finds "transaction declined" even though they share no words. Powered by embedding models + vector databases.
- **它实际上的含义：** 按含义找文档，而不是按关键词匹配。把查询和所有文档嵌入到同一个向量空间，然后返回 embedding 最接近查询 embedding 的文档。“payment failed” 可以找到 “transaction declined”，即使它们没有共享单词。它由 embedding model 和 vector database 支撑。

### Streaming

流式输出
- **What people say:** "Seeing the response appear word by word"
- **人们常说：**“看着回答一个词一个词冒出来”
- **What it actually means:** The LLM sends tokens as they are generated rather than waiting for the complete response. Uses Server-Sent Events (SSE) or WebSocket protocols. Reduces perceived latency from seconds to milliseconds for the first token. Essential for production chat interfaces. Each chunk contains a delta (partial token or word).
- **它实际上的含义：** LLM 在生成 token 时就立即发送，而不是等完整回答生成完。它使用 Server-Sent Events，也就是 SSE，或 WebSocket 协议。它能把首个 token 的感知延迟从秒级降到毫秒级。生产聊天界面离不开它。每个 chunk 都包含一个 delta，也就是部分 token 或词。

### Self-Attention

自注意力
- **What people say:** "How the model decides what to focus on"
- **人们常说：**“模型决定关注什么的方式”
- **What it actually means:** Each token computes query, key, and value vectors. Attention weight between two tokens = dot product of their query and key, scaled and softmaxed. Output = weighted sum of value vectors. Lets every token see every other token.
- **它实际上的含义：** 每个 token 都会计算 query、key 和 value 向量。两个 token 之间的 attention weight 等于 query 和 key 的点积，再缩放并 softmax。输出是 value 向量的加权和。它让每个 token 都能看到其他所有 token。

### SFT (Supervised Fine-Tuning)

监督微调
- **What people say:** "Teaching the model to follow instructions"
- **人们常说：**“教模型听指令”
- **What it actually means:** Fine-tuning a pre-trained model on (instruction, response) pairs. The model learns to generate the response given the instruction. This is what turns a base model into a chat model.
- **它实际上的含义：** 在 instruction 和 response 配对数据上微调预训练模型。模型学习在给定指令时生成对应回答。这就是把 base model 变成 chat model 的关键步骤。

### Softmax

Softmax
- **What people say:** "Turns numbers into probabilities"
- **人们常说：**“把数字变成概率”
- **What it actually means:** softmax(x_i) = exp(x_i) / sum(exp(x_j)). Transforms a vector of arbitrary real numbers into a probability distribution (all positive, sums to 1). Used in classification heads, attention weights, and anywhere you need probabilities.
- **它实际上的含义：** softmax(x_i) = exp(x_i) / sum(exp(x_j))。它把任意实数向量转换成概率分布，所有值为正且总和为 1。它用于分类头、attention weights，以及任何需要概率的地方。

### Swarm

群体智能
- **What people say:** "A bunch of AI agents working together like bees"
- **人们常说：**“一群 AI agent 像蜜蜂一样协作”
- **What it actually means:** Multiple agents sharing state and coordinating through message passing, with emergent behavior arising from simple individual rules rather than central control
- **它实际上的含义：** 多个 agent 共享状态，并通过消息传递协作。它的涌现行为来自简单个体规则，而不是中央控制

## T

### System Prompt

系统提示
- **What people say:** "The AI's instructions"
- **人们常说：**“AI 的指令”
- **What it actually means:** A special message at the start of a conversation that sets the model's behavior, persona, and constraints. Processed before user messages. Not visible to the user in most UIs. Defines what the model should and shouldn't do, its tone, format preferences, and domain focus. Different from user prompts -- system prompts are set by the developer.
- **它实际上的含义：** 对话开始处的一条特殊消息，用来设定模型行为、角色和约束。它在用户消息之前处理。多数 UI 中用户看不到它。它定义模型该做什么、不该做什么、语气、格式偏好和领域重点。它不同于 user prompt，system prompt 由开发者设置。

### Tensor

张量
- **What people say:** "A multi-dimensional array"
- **人们常说：**“多维数组”
- **What it actually means:** The fundamental data structure in deep learning frameworks. A 0D tensor is a scalar, 1D is a vector, 2D is a matrix, 3D+ is a tensor. In PyTorch and JAX, tensors track their computation history for automatic differentiation and can live on CPU or GPU. All neural network inputs, outputs, weights, and gradients are tensors.
- **它实际上的含义：** 深度学习框架中的基础数据结构。0D tensor 是标量，1D 是向量，2D 是矩阵，3D 及以上通常称为张量。在 PyTorch 和 JAX 中，tensor 会跟踪计算历史以支持自动微分，并且可以放在 CPU 或 GPU 上。神经网络的输入、输出、权重和梯度都是 tensor。

### Token

Token
- **What people say:** "A word"
- **人们常说：**“一个词”
- **What it actually means:** A subword unit (typically 3-4 characters in English) produced by a tokenizer like BPE. "unbelievable" might be 3 tokens: "un" + "believ" + "able"
- **它实际上的含义：** 由 BPE 等 tokenizer 产生的子词单元，英语里通常是 3 到 4 个字符。“unbelievable” 可能被切成 3 个 token：“un” + “believ” + “able”

### Temperature

温度
- **What people say:** "Creativity setting"
- **人们常说：**“创造力设置”
- **What it actually means:** A scalar that divides logits before softmax. Temperature=1 is default. Higher = flatter distribution = more random outputs. Lower = sharper distribution = more deterministic. Temperature=0 is argmax (always pick the most likely token).
- **它实际上的含义：** 一个在 softmax 之前除以 logits 的标量。Temperature=1 是默认值。更高意味着分布更平，输出更随机。更低意味着分布更尖，输出更确定。Temperature=0 等价于 argmax，也就是总选最可能的 token。

### Transfer Learning

迁移学习
- **What people say:** "Using a pre-trained model"
- **人们常说：**“使用预训练模型”
- **What it actually means:** Taking a model trained on one task and adapting it to a different task. The early layers learn general features (edges, syntax patterns) that transfer. Only the later layers need task-specific training. This is why you can fine-tune BERT for any NLP task.
- **它实际上的含义：** 把在一个任务上训练好的模型适配到另一个任务。早期层会学到可迁移的通用特征，比如边缘和语法模式。只有后面层需要任务专用训练。这就是为什么你可以把 BERT 微调到各种 NLP 任务上。

### Transformer

Transformer
- **What people say:** "The architecture behind modern AI"
- **人们常说：**“现代 AI 背后的架构”
- **What it actually means:** A neural network architecture that processes sequences using self-attention (letting every position attend to every other position) instead of recurrence, enabling massive parallelization
- **它实际上的含义：** 一种神经网络架构，用 self-attention 处理序列，让每个位置关注其他所有位置，而不是用循环结构，因此可以大规模并行化
- **Why it's called that:** It transforms input representations into output representations through attention layers
- **为什么这样命名：** 它通过 attention 层把输入表示转换成输出表示

## U

### Underfitting

欠拟合
- **What people say:** "The model isn't learning"
- **人们常说：**“模型没学会”
- **What it actually means:** The model is too simple to capture the patterns in the data. Training loss stays high. Fix with: more parameters, more layers, longer training, lower regularization, better features.
- **它实际上的含义：** 模型太简单，无法捕捉数据中的模式。训练损失一直很高。解决方法包括增加参数、增加层数、训练更久、降低正则化、改进特征。

## V

### VAE (Variational Autoencoder)

变分自编码器
- **What people say:** "A generative model"
- **人们常说：**“一种生成模型”
- **What it actually means:** An autoencoder that learns a smooth latent space by forcing the encoder output to follow a Gaussian distribution. You can sample from this distribution and decode to generate new data. The reparameterization trick makes it trainable via backpropagation.
- **它实际上的含义：** 一种 autoencoder，通过迫使 encoder 输出服从高斯分布来学习平滑的 latent space。你可以从这个分布采样，再 decode 生成新数据。重参数化技巧让它可以通过反向传播训练。

### Vector Database

向量数据库
- **What people say:** "A special database for AI"
- **人们常说：**“AI 专用数据库”
- **What it actually means:** A database optimized for storing vectors (dense arrays of floats) and performing fast approximate nearest-neighbor search. The core operation in similarity search, RAG, and recommendation systems.
- **它实际上的含义：** 一种为存储向量，也就是稠密浮点数组，并执行快速近似最近邻搜索而优化的数据库。它是相似度搜索、RAG 和推荐系统中的核心操作。

## W

### Weight

权重
- **What people say:** "What the model learned"
- **人们常说：**“模型学到的东西”
- **What it actually means:** A single number in a model's parameter matrix. A linear layer with input size 768 and output size 3072 has 768*3072 = 2,359,296 weights. Training adjusts each weight to minimize the loss function.
- **它实际上的含义：** 模型参数矩阵中的一个数字。输入大小 768、输出大小 3072 的线性层有 768*3072 = 2,359,296 个权重。训练会调整每个权重，以最小化损失函数。

### Weight Decay

权重衰减
- **What people say:** "Regularization"
- **人们常说：**“正则化”
- **What it actually means:** Adding a penalty proportional to the magnitude of weights to the loss function. Equivalent to L2 regularization. Prevents weights from growing too large. Typical value: 0.01-0.1.
- **它实际上的含义：** 在损失函数中加入一个与权重大小成比例的惩罚项。它等价于 L2 正则化，可以防止权重变得过大。典型值是 0.01 到 0.1。

## Z

### Zero-Shot

零样本
- **What people say:** "No training needed"
- **人们常说：**“不需要训练”
- **What it actually means:** Using a model on a task it wasn't explicitly trained for, with no task-specific examples in the prompt. The model generalizes from pre-training. Works because large models have seen enough variety to handle new task formats.
- **它实际上的含义：** 在 prompt 中不给任务专用示例，直接把模型用于它没有明确训练过的任务。模型从预训练中泛化。大模型见过足够多样的模式，所以能处理新的任务格式。
