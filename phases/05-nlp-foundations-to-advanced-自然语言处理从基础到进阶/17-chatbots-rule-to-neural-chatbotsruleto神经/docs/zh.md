# 聊天机器人，从规则系统到神经网络再到 LLM 智能体

> 说明：ELIZA replied 使用 pattern matches. DialogFlow mapped intents. GPT answered 从 weights. Claude runs tools 与 verifies. Each era solved the previous one's worst failure.

**类型:** 学习
**语言:** Python
**先修要求:** 说明：Phase 5 · 13 (Question Answering), Phase 5 · 14 (Information Retrieval)
**时间:** 约75分钟

## 问题

一个 用户 says "I want to change my flight." The 系统 has to figure out what they want, what information is missing, how to get it, 与 how to complete the action. Then the 用户 says "wait, what if I cancel instead?" 与 the 系统 has to remember the context, switch tasks, 与 preserve 状态.

Conversation is hard 面向 an ML 系统. The input is 开放-ended. The 输出 has to be coherent over many turns. The 系统 may need to act on the world (change a flight, charge a card). Every 错误 step is visible to the 用户.

说明：Chatbot architectures have cycled through four paradigms, each introduced 因为 the previous one failed too visibly. This lesson walks them in order. The 2026 生产 landscape is a hybrid of the last two.

## 概念

![说明：Chatbot evolution: rule-based → 检索 → neural → agent](../assets/chatbot.svg)

**Rule-based (ELIZA, AIML, DialogFlow).** Hand-authored patterns match 用户 input 与 produce responses. Intent classifiers route to predefined flows. Slot-filling 状态 machines collect required info. Works brilliantly inside the narrow scope it was designed 面向. Fails immediately outside it. Still ships in safety-critical domains (banking authentication, airline booking) 其中 hallucination is not tolerated.

**Retrieval-based.** A FAQ-style 系统. Encode every pair of (utterance, response). At runtime, encode the 用户's message 与 retrieve the nearest stored response. Think Zendesk's 经典 "similar articles" feature. Handles paraphrases better than rules. No 生成, so no hallucination.

说明：**Neural (seq2seq).** Encoder-decoder trained on conversation logs. Generates responses 从 scratch. Fluent but prone to generic outputs ("I don't know") 与 factual drift. Never reliably on 主题. The 原因 Google, Facebook, 与 Microsoft all had disappointing chatbots in 2016-2019.

**LLM agents.** A 语言模型 wrapped in a loop 这 plans, calls tools, 与 verifies outcomes. Not a 聊天机器人 使用 a 长 prompt. An agent loop: plan → call tool → observe result → decide next step. Retrieval-first grounding (RAG) keeps it 从 hallucinating. Tool calls let it actually do things. This is the 2026 architecture.

这个 four paradigms are not sequential replacements. A 2026 生产 聊天机器人 routes through all four: rule-based 面向 authentication 与 destructive actions, 检索 面向 FAQ, neural 生成 面向 natural phrasing, LLM agent 面向 ambiguous 开放-ended queries.

## 动手构建

### Step 1: rule-based pattern matching

```python
import re


class RulePattern:
    def __init__(self, pattern, response_template):
        self.regex = re.compile(pattern, re.IGNORECASE)
        self.template = response_template


PATTERNS = [
    RulePattern(r"my name is (\w+)", "Nice to meet you, {0}."),
    RulePattern(r"i (need|want) (.+)", "Why do you {0} {1}?"),
    RulePattern(r"i feel (.+)", "Why do you feel {0}?"),
    RulePattern(r"(.*)", "Tell me more about that."),
]


def rule_based_respond(user_input):
    for pattern in PATTERNS:
        m = pattern.regex.match(user_input.strip())
        if m:
            return pattern.template.format(*m.groups())
    return "I don't understand."
```

说明：ELIZA in 20 lines. The reflection trick ("I feel sad" → "Why do you feel sad") is the canonical psychotherapist demo 从 Weizenbaum 1966. Still instructive.

### Step 2: 检索-based (FAQ)

This illustrative snippet requires `pip install sentence-transformers` (这 pulls in torch). The runnable `code/main.py` 面向 this lesson uses a stdlib Jaccard 相似度 instead, so the lesson runs 不使用 external dependencies.

```python
from sentence_transformers import SentenceTransformer
import numpy as np


FAQ = [
    ("how do i reset my password", "Go to Settings > Security > Reset Password."),
    ("how do i cancel my order", "Go to Orders, find the order, click Cancel."),
    ("what is your return policy", "30-day returns on unused items, original packaging."),
]


encoder = SentenceTransformer("sentence-transformers/all-MiniLM-L6-v2")
faq_questions = [q for q, _ in FAQ]
faq_embeddings = encoder.encode(faq_questions, normalize_embeddings=True)


def faq_respond(user_input, threshold=0.5):
    q_emb = encoder.encode([user_input], normalize_embeddings=True)[0]
    sims = faq_embeddings @ q_emb
    best = int(np.argmax(sims))
    if sims[best] < threshold:
        return None
    return FAQ[best][1]
```

说明：Threshold-based refusal is the key design choice. If the best match is not close enough, return `None` 与 let the 系统 escalate.

### Step 3: neural 生成 (基线)

Use a 小 instruction-tuned encoder-decoder (FLAN-T5) 或 a fine-tuned conversational 模型. Production-unusable on its own in 2026 (contradiction, off-主题 drift, factual nonsense), but ships inside hybrid systems 面向 natural phrasing. DialoGPT-style decoder-only models need explicit turn separators 与 EOS handling to produce coherent replies; a FLAN-T5 text2text 流水线 works out of the box 面向 a teaching example.

```python
from transformers import pipeline

chatbot = pipeline("text2text-generation", model="google/flan-t5-small")

response = chatbot("Respond politely to: Hi there!", max_new_tokens=40)
print(response[0]["generated_text"])
```

### Step 4: LLM agent loop

这个 2026 生产 shape:

```python
def agent_loop(user_message, tools, llm, max_steps=5):
    history = [{"role": "user", "content": user_message}]
    for _ in range(max_steps):
        response = llm(history, tools=tools)
        tool_call = response.get("tool_call")
        if tool_call:
            tool_name = tool_call.get("name")
            args = tool_call.get("arguments")
            if not isinstance(tool_name, str) or tool_name not in tools:
                history.append({"role": "assistant", "tool_call": tool_call})
                history.append({"role": "tool", "name": str(tool_name), "content": f"error: unknown tool {tool_name!r}"})
                continue
            if not isinstance(args, dict):
                history.append({"role": "assistant", "tool_call": tool_call})
                history.append({"role": "tool", "name": tool_name, "content": f"error: arguments must be a dict, got {type(args).__name__}"})
                continue
            fn = tools[tool_name]
            result = fn(**args)
            history.append({"role": "assistant", "tool_call": tool_call})
            history.append({"role": "tool", "name": tool_name, "content": result})
        else:
            return response["content"]
    return "I could not complete the task in the step budget."
```

说明：Three things to name. Tools are callable functions the LLM can invoke. The loop terminates when the LLM returns a final 答案 instead of a tool call. The step 预算 prevents infinite loops on ambiguous tasks.

Real 生产 adds: 检索-first grounding (inject relevant docs before each LLM call), guardrails (refuse destructive actions 不使用 confirmation), observability (log every step), 与 evaluations (automated checks 这 agent behavior stays on-spec).

### Step 5: hybrid routing

```python
def hybrid_chat(user_input):
    if is_destructive_action(user_input):
        return structured_flow(user_input)

    faq_answer = faq_respond(user_input, threshold=0.6)
    if faq_answer:
        return faq_answer

    return agent_loop(user_input, tools, llm)


def is_destructive_action(text):
    danger_words = ["delete", "cancel", "charge", "refund", "transfer"]
    return any(w in text.lower() for w in danger_words)
```

这个 pattern: 确定性 rules 面向 anything destructive, 检索 面向 canned FAQs, LLM agents 面向 everything else. This is what ships in 2026 customer-support systems.

## 投入使用

这个 2026 stack:

|Use case|Architecture|
|---------|---------------|
|Booking, payment, authentication|Rule-based 状态 machines + 槽位 filling|
|Customer support FAQs|Retrieval over curated answers|
|开放-ended help chat|LLM agent 使用 RAG + tool calls|
|Internal tools / IDE assistants|LLM agent 使用 tool calls (search, read, write)|
|Companion / character chatbots|Tuned LLM 使用 persona 系统 prompt, 检索 on knowledge|

说明：Always use hybrid routing in 生产. No single architecture handles every request well. The routing layer itself is typically a 小 intent classifier.

## Failure modes 这 still ship

- 说明：**Confident fabrication.** LLM agent claims it completed an action it did not. Mitigation: verify outcomes, log tool calls, never let the LLM claim to have done something 不使用 a successful tool return.
- **Prompt injection.** 用户 inserts 文本 这 overrides the 系统 prompt. Ranked LLM01 in the OWASP Top 10 面向 LLM Applications 2025. Two flavors: direct injection (pasted 到 the chat) 与 indirect injection (hidden in 文档, emails, 或 tool outputs the agent reads).

说明：  Attack rates vary by scenario. Measured success rates range ~0.5-8.5% across frontier models in general tool-use 与 coding benchmarks. Specific high-risk setups (adaptive attacks against AI coding agents, vulnerable orchestration) have reached ~84%. Production CVEs include EchoLeak (CVE-2025-32711, CVSS 9.3)，a zero-click 数据-exfiltration flaw in Microsoft 365 Copilot triggered by an attacker-controlled email.

  Mitigations: treat 用户 input as untrusted throughout the loop; sanitize before tool calls; isolate tool outputs 从 the main prompt; use the Plan-Verify-Execute (PVE) pattern 其中 the agent plans first, then verifies each action against 这 plan before executing (this stops tool results 从 injecting new unplanned actions); require 用户 confirmation 面向 destructive actions; apply least-privilege to tool scopes.

说明：  No amount of prompt engineering fully eliminates this risk. External runtime defense layers (LLM Guard, allowlist validation, 语义 anomaly detection) are required.
- **Scope creep.** Agent goes off-任务 因为 a tool call returned tangentially related info. Mitigation: narrow tool contracts; keep the 系统 prompt focused; add evaluations 面向 off-任务 rate.
- 说明：**Infinite loops.** Agent keeps calling the same tool. Mitigation: step 预算, tool-call deduplication, LLM judge on "are we making progress."
- **Context window exhaustion.** 长 conversations push the earliest turns out of context. Mitigation: summarize older turns, retrieve relevant past turns by 相似度, 或 use a 长-context 模型.

## 交付成果

保存为 `outputs/skill-chatbot-architect.md`:

```markdown
---
name: chatbot-architect
description: Design a chatbot stack for a given use case.
version: 1.0.0
phase: 5
lesson: 17
tags: [nlp, agents, chatbot]
---

Given a product context (user need, compliance constraints, available tools, data volume), output:

1. Architecture. Rule-based, retrieval, neural, LLM agent, or hybrid (specify which paths go where).
2. LLM choice if applicable. Name the model family (Claude, GPT-4, Llama-3.1, Mixtral). Match to tool-use quality and cost.
3. Grounding strategy. RAG sources, retrieval method (see lesson 14), tool contracts.
4. Evaluation plan. Task success rate, tool-call correctness, off-task rate, hallucination rate on held-out dialogs.

Refuse to recommend a pure-LLM agent for any destructive action (payments, account deletion, data modification) without a structured confirmation flow. Refuse to skip the prompt-injection audit if the agent has write access to anything.
```

## 练习

1. 说明：**Easy.** Implement the rule-based respond above 使用 10 patterns 面向 a coffee-shop ordering bot. Test 边缘 cases: double orders, modifications, cancellation, unclear intent.
2. **Medium.** Build a hybrid FAQ + LLM fallback. 50 canned FAQ entries 面向 a SaaS product, LLM fallback 使用 检索 over the docs site. Measure refusal rate 与 准确率 on 100 real support questions.
3. **Hard.** Implement the agent loop above 使用 three tools (search, read-用户-数据, send-email). Run an 评估 使用 50 test scenarios including prompt injection attempts. Report off-任务 rate, failed 任务 rate, 与 any injection success.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|Intent|What the 用户 wants|说明：Categorical 标签 (book_flight, reset_password). Routed to a handler.|
|Slot|一个 piece of info|说明：Parameter the bot needs (date, destination). Slot filling is the 序列 of asks.|
|RAG|Retrieval plus 生成|说明：Retrieve relevant docs, then ground the LLM's response.|
|Tool call|Function invocation|说明：LLM emits a structured call 使用 name + args. Runtime executes, returns result.|
|Agent loop|Plan, act, verify|说明：Controller 这 runs LLM calls interleaved 使用 tool calls until 任务 complete.|
|Prompt injection|用户 attacks prompt|说明：Malicious input 这 tries to override the 系统 prompt.|

## 延伸阅读

- [说明：Weizenbaum (1966). ELIZA，A Computer Program 面向 the Study of Natural Language Communication](https://web.stanford.edu/class/cs124/p36-weizenabaum.pdf)，the original rule-based 聊天机器人 paper.
- [说明：Thoppilan et al. (2022). LaMDA: Language Models 面向 Dialog Applications](https://arxiv.org/abs/2201.08239)，Google's late neural-聊天机器人 paper, just before LLM agents took over.
- 说明：[说明：Yao et al. (2022). ReAct: Synergizing Reasoning 与 Acting in Language Models](https://arxiv.org/abs/2210.03629)，the paper 这 named the agent loop pattern.
- 说明：[说明：Anthropic's guide on building effective agents](https://www.anthropic.com/research/building-effective-agents)，2024 生产 guidance 这 still holds in 2026.
- 说明：[说明：Greshake et al. (2023). Not what you've signed up 面向: Compromising Real-World LLM-Integrated Applications 使用 Indirect Prompt Injection](https://arxiv.org/abs/2302.12173)，the prompt-injection paper.
- 说明：[说明：OWASP Top 10 面向 LLM Applications 2025，LLM01 Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)，the ranking 这 made prompt injection the top security concern.
- 说明：[说明：AWS，Securing Amazon Bedrock Agents against Indirect Prompt Injections](https://aws.amazon.com/blogs/machine-learning/securing-amazon-bedrock-agents-a-guide-to-safeguarding-against-indirect-prompt-injections/)，practical orchestration-layer defenses including Plan-Verify-Execute 与 用户-confirmation flows.
- 说明：[EchoLeak (CVE-2025-32711)](https://www.vectra.ai/topics/prompt-injection)，the canonical zero-click 数据-exfiltration CVE 从 indirect prompt injection. Reference case 面向 why write-access agents need runtime defenses.
