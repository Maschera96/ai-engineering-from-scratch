# 具身 VLA：RT-2、OpenVLA、π0、GR00T

> 模型第一次从网页上读取菜谱并在厨房机器人上执行，是 RT-2（Google DeepMind，2023 年 7 月）。RT-2 将动作离散化为文本 token，在网络数据加机器人动作数据上对一个 VLM 做联合微调，并证明了网络规模的视觉-语言知识可以迁移到机器人控制上。OpenVLA（2024 年 6 月）发布了开源的 7B 参考实现。Physical Intelligence 的 π0 系列（2024-2025）加入了流匹配动作专家。NVIDIA 的 GR00T N1（2025 年 3 月）为人形机器人大规模交付了双系统（System 1 / System 2）控制。VLA 原语——vision-language-action，一个能看、能读、能行动的单一模型——是连接本阶段理解模型与第 15 阶段自主系统的桥梁。

**类型：** 学习
**语言：** Python（标准库，动作 tokenizer + VLA 推理骨架）
**前置条件：** 阶段 12 · 05（LLaVA）、阶段 15（自主系统，引用）
**时长：** ~180 分钟

## 学习目标

- 描述动作 tokenization：离散分桶编码（RT-2）、FAST 高效动作 token、连续流匹配动作（π0）。
- 解释为何在网络数据 + 机器人数据上联合微调能保留通用知识向新任务的迁移能力。
- 在同一个机器人任务上比较 OpenVLA（开源 7B Llama+VLM）、π0（流匹配）和 GR00T N1（双系统）。
- 说出 Open X-Embodiment 数据集及其作为 RT-X 训练语料的角色。

## 问题

让机器人按自然语言指令做家务，自 1970 年代起就是一个研究目标。2020 年代的答案：vision-language-action（VLA）模型。与用于 VQA 的 VLM 架构相同，但输出是动作（关节力矩、末端执行器位姿、离散命令）而非文本。

VLA 特有的挑战：

1. 动作空间是连续的（关节角度、力）且高维（7 自由度机械臂 + 3 自由度夹爪 = 30 Hz 下的 10 维）。
2. 机器人专用训练数据稀缺。Open X-Embodiment 有约 100 万条轨迹；网络文本-图像有 50 亿+。
3. 控制频率很重要。30 Hz 控制回路意味着每个动作的预算为 33ms。
4. 安全。错误的动作会损坏硬件、伤害人或破坏财物。

## 概念

### 动作 tokenization（RT-2）

RT-2 的诀窍：把每个关节目标表示为一个量化的文本 token。将归一化的 [-1, 1] 区间离散化为 256 个桶，把每个桶映射到一个词表 ID。一个 10 自由度的动作在每个控制步变成 10 个 token。

在一个混合数据上对 PaLM-X VLM 做联合微调：

- 网络图像-文本对（caption、VQA）。
- 机器人示范，动作作为 token。

模型看到 "pick up the red cube"（语言）→ 图像（视觉）→ 10-token 动作序列（离散化的关节目标）。网络预训练保留了通用知识迁移：即便 "fast-moving" 不在训练数据中，RT-2 也能执行 "move towards the fast-moving object"。

RT-2 论文中的推理速度为 3-5 Hz，受限于 VLM 自回归解码。

### OpenVLA——开源 7B 参考实现

OpenVLA（Kim 等人，2024 年 6 月）是 RT-2 的开源权重对等物。7B Llama 主干，DINOv2 + SigLIP 双视觉编码器，在 256 个桶上做动作 tokenization。

在 Open X-Embodiment（横跨 22 种机器人的 97 万条轨迹）上训练。配套提供 LoRA 微调支持，用于适配新机器人。

推理：在 A100 上量化后达到 4-5 Hz。对于慢速操作够用，但不足以做高频控制。

### FAST tokenizer——更快的动作解码

Pertsch 等人（2024）指出离散分桶 tokenization 效率低下——大多数动作聚集在桶空间的一个小区域。FAST（Frequency-domain Action Sequence Tokenizer）通过 DCT 压缩动作序列并量化系数。

一个 30 步的动作轨迹变成约 10 个 FAST token，而非 300 个离散分桶 token。推理速度提升 3-5 倍，且无质量损失。

### π0 与流匹配动作

Physical Intelligence 的 π0（Black 等人，2024 年 10 月）用一个流匹配动作专家替代离散动作 token：

- 一个小型动作 transformer 读取 VLM 的隐藏状态，通过 rectified flow 输出一个连续的 50 步动作序列。
- 动作头用流匹配损失训练；VLM 预训练保持不变。
- 推理：整个动作序列在约 5 个去噪步内发出，实际控制频率达 50 Hz。

π0 的论断：在广泛的操作任务套件上击败 OpenVLA 和 Octo。连续动作的表述保留了离散化所破坏的平滑性。

π0.5 和 π0-FAST 是渐进式升级。π0-FAST 将 FAST tokenization 与流匹配结合。

### GR00T N1——人形机器人的双系统

NVIDIA 的 GR00T N1（2025 年 3 月）为人形机器人（>30 自由度，全身）打造：

- System 2：一个大型 VLM，读取场景 + 指令，以约 1 Hz 产生高层子目标。
- System 1：一个小型动作头 transformer，以子目标为条件产生 50-100 Hz 的低层关节命令。

这种拆分对应卡尼曼的快思考与慢思考：System 2 规划，System 1 行动。好处：缓慢的 VLM 规模规划不会阻塞快速控制；System 1 保持小型以降低延迟。

GR00T N1.7（2025 年末）改进了数据扩展。GR00T 用来自 Omniverse 的 sim-to-real 数据进行微调。

### Open X-Embodiment

训练数据。RT-X（2023 年 10 月）汇集了 22 个数据集，覆盖横跨 22 种机器人的 100 万条轨迹。Open X-Embodiment 是大家都在用的语料：

- ALOHA / Bridge V2 / Droid / RT-2 Kitchen / Language Table。
- 每个样本：（机器人状态、相机视角、指令、动作序列）。
- 训练规范：统一动作空间、归一化关节范围、调整相机尺寸。

OpenVLA 和 π0 在 Open X-Embodiment 上训练。与任何特定机器人之间的领域差距通过在 100-1000 个任务专用示范上做 LoRA 微调来弥合。

### 联合微调 vs 仅机器人数据

联合微调将网络 VQA 数据与机器人轨迹混合。比例很重要：VQA 太多模型会忘记动作；机器人数据太多模型会丧失通用知识。

RT-2 的比例：约 1:1。OpenVLA：网络对机器人约 0.5:1。π0：类似。具体比例是一个需按数据集规模调整的超参数。

仅用机器人数据训练会产生任务专用模型，在分布外指令上失败。联合微调正是 "pick up the red cube（示范中的）" 与 "pick up the third largest object from the left（新颖措辞）" 之间的差别。

### 安全与动作限制

每个生产级 VLA 都配套：

- 硬关节限位（不能超出规格施加力矩）。
- 速度限制（软裁剪）。
- 工作空间边界（末端执行器不能离开桌面）。
- 对新颖任务的人在环审批。

这些位于 VLA 之外，作为控制层检查。VLA 的输出是建议，而非命令。

## 动手用

`code/main.py`：

- 实现 256 桶动作 tokenization 与去 tokenization。
- 勾勒一个基于 DCT + 量化的 FAST tokenizer。
- 比较（离散分桶、FAST、连续流）三者每个动作步的 token 数。
- 打印 RT-2 → OpenVLA → π0 → GR00T 的谱系摘要。

## 交付

本课产出 `outputs/skill-vla-action-format-picker.md`。给定一个机器人任务（操作、导航、人形全身），在离散分桶 + RT-2、FAST + OpenVLA、流匹配 + π0、或双系统 + GR00T 之间做选择。

## 练习

1. 一个 10 自由度机械臂，控制速率 30 Hz。256 桶的离散分桶 tokenization 每秒发出多少 token？一个 7B VLM 跟得上吗？

2. FAST tokenization 将 30 步轨迹压缩到约 10 个 token。如果轨迹包含高频运动（例如击鼓），用户会损失什么？

3. π0 的流匹配头在约 5 步内完成去噪。将其吞吐量与 OpenVLA 4-5 Hz 的自回归解码做比较。

4. GR00T 的 System 1 / System 2 拆分对应卡尼曼。提出一个可能有助于双足行走的不同拆分（System 3？）。

5. 阅读 Open X-Embodiment 第 4 节关于数据集策展的内容。说出防止领域泄漏的三条策展规则。

## 关键术语

| 术语 | 大家怎么说 | 实际含义 |
|------|-----------------|------------------------|
| VLA | "Vision-language-action" | 接收图像 + 指令并输出动作命令的模型 |
| 动作 tokenization | "离散分桶" | 将连续关节目标量化为每维 256 个桶，每个桶一个词表 ID |
| FAST tokenizer | "频域动作 token" | DCT + 量化，将 30 步轨迹压缩到约 10 个 token |
| 联合微调 | "混合网络 + 机器人" | 在网络 VQA 数据与机器人示范上一起训练，以保留通用知识 |
| 流匹配动作头 | "π0 连续输出" | 通过 rectified flow 输出 50 步动作序列的小型 transformer |
| System 1 / System 2 | "双系统控制" | 大型 VLM 慢速规划，小型动作头快速行动；GR00T 模式 |
| Open X-Embodiment | "RT-X 数据集" | 100 万条轨迹的跨机器人数据集；训练语料 |

## 延伸阅读

- [Brohan et al. — RT-2 (arXiv:2307.15818)](https://arxiv.org/abs/2307.15818)
- [Kim et al. — OpenVLA (arXiv:2406.09246)](https://arxiv.org/abs/2406.09246)
- [Black et al. — π0 (arXiv:2410.24164)](https://arxiv.org/abs/2410.24164)
- [NVIDIA — GR00T N1 (arXiv:2503.14734)](https://arxiv.org/abs/2503.14734)
- [Open X-Embodiment Collab — RT-X (arXiv:2310.08864)](https://arxiv.org/abs/2310.08864)
