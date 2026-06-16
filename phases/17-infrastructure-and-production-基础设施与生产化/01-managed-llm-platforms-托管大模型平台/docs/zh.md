# 托管大模型平台：Bedrock、Vertex AI、Azure OpenAI

> Three hyperscalers, three distinct strategies. AWS Bedrock is 一个模型市场 ： Claude, Llama, Titan, Stability, Cohere behind one API. Azure OpenAI is an exclusive OpenAI partnership plus 预置吞吐量单元 (PTUs) for 专用容量. Vertex AI is Gemini 优先 with the best 长上下文 和 多模态 story. In 2026 Artificial Analysis measures Azure OpenAI at ~50 ms 中位数 and Bedrock at ~75 ms on Llama 3.1 405B equivalents ： PTUs 解释 the gap because 专用容量 beats 共享按需容量. 这个决策 rule is not "which is fastest" but "which 模型 目录 and FinOps 界面 match my 产品." This lesson teaches you to pick with the tradeoffs written down, not vibes.

**Type:** Learn
**Languages:** Python（标准库， 玩具 cost-and-latency比较器)
**Prerequisites:** Phase 11 (大模型工程), Phase 13 (工具与协议)
**Time:** ~60 分钟

## 学习目标

- Name the three 平台 strategies (marketplace 对比 exclusive 对比 Gemini 优先) and match each to 一个产品 use case.
- 解释 什么 预置吞吐量单元 (PTUs) buy you in Azure OpenAI and why 按需 Bedrock typically reads ~25 ms slower at the 405B scale.
- Diagram the FinOps attribution 界面 for each 平台 (Bedrock Application Inference 画像 对比 Vertex project-per-团队 对比 Azure scopes + PTU reservations).
- Write down 一个"双提供商最低要求" 策略 和 解释 为什么 单供应商锁定 is the expensive mistake in 2026.

## 问题

You picked Claude 3.7 Sonnet for your 产品. Now you need to serve it. You can call the Anthropic API 直接, or you can call it through AWS Bedrock, or you can go through 一个网关. The direct API is 这个最简单; Bedrock adds BAAs, VPC endpoints, IAM, and CloudWatch attribution. 这个网关 adds 故障切换, 统一 billing, 和 限流 跨 提供商.

The deeper question is 目录. If you need Claude and Llama and Gemini in the same 产品, you cannot buy them all from one place unless that place is Bedrock plus Vertex plus Azure OpenAI 同时. The hyperscalers are not interchangeable ： they each made a different bet on who owns 这个模型 layer.

This lesson maps the three bets, 这个延迟 gap, the FinOps gap, and the lock-in 风险.

## 概念

### 三种策略

**AWS Bedrock** ： the marketplace. Claude (Anthropic), Llama (Meta), Titan (AWS first-party), Stability (image), Cohere (embeddings), Mistral, plus image and embedding sub-目录. One API, one IAM 界面, one CloudWatch 导出. Bedrock's bet is that customers want optionality more than they want a single 模型.

**Azure OpenAI** ： 这个独家合作. You get GPT-4 / 4o / 5 / o-series, DALL·E, Whisper, and fine-tuning of OpenAI 模型 in Azure datacenters. No non-OpenAI 模型 in 这个"Azure OpenAI Service" 目录 ： those go to Azure AI Foundry (separate 产品). Azure's bet is that OpenAI remains the frontier and customers want 企业级 控制 on that 具体关系.

**Vertex AI** ： Gemini first, everything else second. Gemini 1.5 / 2.0 / 2.5 Flash and Pro, plus 模型 Garden (third-party). Vertex's bet is 多模态 长上下文 ： 1M-词元 Gemini context is 这个差异点.

### 规模化时的延迟差距

Artificial Analysis runs 连续 基准. On equivalent Llama 3.1 405B deployments (共享按需容量), Azure OpenAI 中位数 首词元延迟 is around 50 ms; Bedrock is around 75 ms. The gap is not an AWS failure ： it is 一个容量 模型 difference. Azure sells PTUs (预置吞吐量单元), which reserve GPU 容量 for your tenant. Bedrock's equivalent (预置吞吐量) exists but starts around $21/小时 per unit, and most customers stay on 共享按需容量.

按需 shared 容量 competes with every other 客户's 流量. 专用容量 does not. If your 产品 SLA is TTFT < 100 ms at P99, you either buy PTUs on Azure, buy Bedrock 预置吞吐量, or accept 这个默认 方差.

### 预置吞吐量经济学

Azure PTUs: 一个预留 block of inference compute. Up to ~70% 节省 对比 按需 for predictable 工作负载. Costs fixed per 小时 regardless of 流量 ： you pay for 这个预留 even when 空闲. 这个盈亏平衡 is 通常 around 40-60% 持续利用率.

Bedrock 预置吞吐量: $21-$50 per 小时 depending on 模型 和 区域. 类似 math ： 盈亏平衡 is around half peak utilization. Monthly commitment 必需.

Vertex provisioned 容量 is sold per Gemini SKU; 定价 变化 by 模型 和 区域 and is less publicly advertised.

### FinOps 界面：真正的差异点

**Bedrock Application Inference 画像** are the cleanest attribution in the marketplace. Tag 一个画像 搭配 `team`, `product`, `feature`; 路由 all 模型 invocations through it; CloudWatch breaks out 成本 per 画像 without post-processing. 新增 2025, still the most 细粒度 hyperscaler native.

**Vertex** attribution is project-per-团队 plus 标签-everywhere. You 模型 each 团队 as a GCP project, put 标签 on every resource, and use BigQuery Billing 导出 + DataStudio for 汇总. More work, but BigQuery gives you arbitrary SQL on 这个成本 data.

**Azure** relies on subscription/resource-group scopes plus 标签, with PTU reservations as a first-class 成本 object. 标签 are inherited from resource groups, not 请求, so per-请求 attribution 需要 Application Insights custom 指标 or 一个网关 that stamps 头.

这个模式: Bedrock is cleanest native, Vertex is most 灵活 via BigQuery, Azure is most 不透明 unless you instrument.

### 锁定是 2026 年的风险

Single-hyperscaler commitment was fine when one 模型 dominated. In 2026 the frontier moves 每月： Claude 3.7 one quarter, Gemini 2.5 the next, GPT-5 the quarter after. Locking to one 平台 locks you out of two-thirds of the frontier.

这个模式 working 团队 采用: 双提供商最低要求 for any 产品-关键 LLM call. Bedrock plus Azure OpenAI is the common 组合 ： Claude from one, GPT from the other, 故障切换 between them, same 网关. 成本 uplift is 可忽略，因为网关 routes optimal; availability uplift during 故障 (like the Azure OpenAI January 2025 事故, the AWS us-east-1 outage) is decisive.

### 数据驻留、BAA 与受监管行业

Bedrock: BAAs in most 区域; VPC endpoints; 护栏. Common fintech 默认.
Azure OpenAI: HIPAA, SOC 2, ISO 27001; EU 数据驻留; 这个企业级-regulated 默认.
Vertex: HIPAA, GDPR, 数据驻留 per 区域; Google Cloud's 合规 stack.

All three meet the basic checkbox. The differences are in data 保留策略, how 日志 are 处理, and whether 滥用监控 reads your 流量 (默认 opt-in on most; opt-out 可用 for 企业级).

### 你应该记住的数字

- Azure OpenAI 中位数 TTFT on Llama 3.1 405B equivalents: ~50 ms (with PTUs).
- Bedrock 中位数 TTFT 按需: ~75 ms.
- Bedrock 预置吞吐量: $21-$50/hr per unit.
- Azure PTU 盈亏平衡: ~40-60% 持续利用率.
- PTU 节省 对比 按需 at high utilization: up to 70%.

## 使用它

`code/main.py` compares the three 平台 on 一个合成 工作负载 ： it 模型 按需 对比 PTU economics, TTFT 方差, 和 成本归因 保真度. Run it to see where PTUs pay off and where the marketplace's 模型 breadth outweighs a TTFT gap.

## 交付它

This lesson 产出 `outputs/skill-managed-platform-picker.md`. Given 一个工作负载 画像 (模型 需要, TTFT SLA, daily volume, 合规 要求), it recommends 一个主要平台, a fallback, and a FinOps 埋点 计划.

## 练习

1. Run `code/main.py`. At what 持续利用率 does Azure PTU beat 按需 for a 70B class 模型? Compute 这个盈亏平衡 和 比较 to the advertised 40-60% band.
2. Your 产品 needs Claude 3.7 Sonnet and GPT-4o. Design a two-提供商 部署 ： which goes to which hyperscaler, 什么 网关 sits in front, what is 这个故障切换 策略?
3. A regulated healthcare 客户 需要 BAAs, US-East 数据驻留, and sub-100ms P99 TTFT. Pick 一个平台 和 说明理由 with three 具体功能.
4. You 发现 your Bedrock bill is up 4x this 月 with no 流量 change. Without Application Inference 画像, how would you find 这个元凶? 搭配 画像, how long does it take?
5. Read the Azure OpenAI and Bedrock 定价 pages. For a 100M-词元/月 Claude 工作负载, which is 更便宜 ： direct Anthropic API, Bedrock 按需, or Bedrock 预置吞吐量?

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| Bedrock | "AWS LLM service" | 模型市场 across Claude, Llama, Titan, Mistral, Cohere |
| Azure OpenAI | "Azure's ChatGPT" | Exclusive OpenAI 模型 in Azure datacenters with 企业级 控制 |
| Vertex AI | "Google's LLM" | Gemini 优先 平台 搭配 模型 Garden for third-party 模型 |
| PTU | "专用容量" | 预置吞吐量 Unit ： 预留 inference GPUs, priced per 小时 |
| Application Inference 画像 | "Bedrock tagging" | Per-产品 成本/usage 画像 搭配 标签, CloudWatch-原生|
| 模型 Garden | "Vertex 目录" | Vertex AI's third-party 模型 section, separate from Gemini |
| 双提供商最低要求 | "LLM redundancy" | 策略 of running every 关键 LLM path across ≥2 hyperscalers |
| BAA | "HIPAA paperwork" | Business Associate Agreement; 必需 for PHI; provided by all three |
| Abuse 监控 | "the log watcher" | 提供商-side safety scan on 提示词/输出; opt-out in 企业级 |

## 延伸阅读

- [AWS Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/) ： 权威 rate card and 预置吞吐量 定价.
- [Azure OpenAI Service Pricing](https://azure.microsoft.com/en-us/pricing/details/cognitive-services/openai-service/) ： PTU economics and rate cards.
- [Vertex AI Generative AI Pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing) ： Gemini tiers and 模型 Garden surcharges.
- [Artificial Analysis LLM Leaderboard](https://artificialanalysis.ai/) ： 连续 延迟 和 吞吐量 基准 跨 提供商.
- [The AI Journal — AWS Bedrock vs Azure OpenAI CTO Guide 2026](https://theaijournal.co/2026/03/aws-bedrock-vs-azure-openai/) ： 企业级 决策 framework.
- [Finout — Bedrock vs Vertex vs Azure FinOps](https://www.finout.io/blog/bedrock-vs.-vertex-vs.-azure-cognitive-a-finops-comparison-for-ai-spend) ： attribution mechanics side-by-side.
