---
name: otel-genai-zh
description: 使用 OpenTelemetry GenAI 语义约定为智能体埋点 —— invoke_agent、chat、tool_call span，配以正确的属性和可选启用的内容捕获。
version: 1.0.0
phase: 14
lesson: 23
tags: [opentelemetry, genai, observability, tracing, semantic-conventions]
---

给定一个智能体运行时，接入 OTel GenAI 语义约定。

产出：

1. 每次智能体运行生成一个 `invoke_agent` span。对远程智能体服务使用 CLIENT 类型，对进程内运行使用 INTERNAL。名称：`invoke_agent {gen_ai.agent.name}`。
2. 每次 LLM 调用生成一个 `chat` span，带有 `gen_ai.operation.name=chat`、`gen_ai.provider.name`、`gen_ai.request.model`、`gen_ai.response.model`。
3. 每次工具调用生成一个 `tool_call` span，带有 `gen_ai.tool.name`，并在适用时带有 `gen_ai.data_source.id`（RAG 语料库 / 记忆存储）。
4. 可选启用的内容捕获：默认关闭；开启时，将输入/输出存储在外部，并在 span 上记录 `*.reference_id`。
5. 上下文传播：使用 W3C trace context 头，使多进程运行（Claude Agent SDK CLI 子进程）拼接为一条 trace。

硬性拒绝：

- 默认内联捕获完整的 prompt/输出。存在 PII 与密钥泄露风险；也违反规范。
- 缺少 `gen_ai.provider.name`。多 provider 仪表盘会失效。
- 孤立的工具 span。务必通过活动上下文设置父子关系。

拒绝规则：

- 如果运行时无法跨进程边界传播上下文，拒绝。Claude Agent SDK + CLI 用户需要多进程 trace 拼接。
- 如果产品有合规约束（HIPAA、GDPR），拒绝内联内容捕获。仅允许带访问控制的外部存储。
- 如果后端未设置 `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`，发出警告：属性名可能在 collector 升级时发生变化。

输出：`tracer.py`、`attributes.py`、`content_store.py`、`README.md`，说明 span 结构、稳定性 opt-in 以及内容捕获策略。最后以“接下来读什么”收尾，指向第 24 课（后端：Langfuse、Phoenix、Opik）或第 17 课（Claude Agent SDK trace-context 传播）。
