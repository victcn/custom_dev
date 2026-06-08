# Agent Loop 运行循环

> 参考版本：
> - OpenClaw：`/code/openclaw@ca31a705d02e42ffcfb2c5884bb55339a6d0cbdc`
> - Hermes Agent：`/code/hermes-agent@3d4297a59a8607ed24850524d229f5f42520d087`

## 本章问题

Agent loop 解决的是“一个用户输入如何变成可追踪、可中断、可持久化的长期运行任务”的问题。OpenClaw 文档把一次真实 agent run 定义为 intake、context assembly、model inference、tool execution、streaming replies、persistence 的完整路径，并说明它是保持 session state 一致性的权威路径（文档声明：`/code/openclaw/docs/concepts/agent-loop.md`）。

Hermes 的入口更集中在 Python 类 `AIAgent`：文件头说明 `run_agent.py` 提供带 tool calling 的 standalone agent，并负责 conversation loop、tool execution、response management（源码事实：`/code/hermes-agent/run_agent.py` 顶部模块说明，`class AIAgent`）。

## OpenClaw 怎么做

OpenClaw 的 Gateway RPC 入口是 `agent` 和 `agent.wait`；`agent` 会校验参数、解析 `sessionKey/sessionId`、持久化 session metadata，并立即返回 `{ runId, acceptedAt }`，而不是等待模型跑完（文档声明：`/code/openclaw/docs/concepts/agent-loop.md` 的 `Entry points` 和 `How it works`）。

真正执行 agent 的桥接函数在文档中称为 `agentCommand`：它解析 model、thinking、verbose、trace 默认值，加载 skills snapshot，调用 `runEmbeddedPiAgent`，并在 embedded loop 没有发出终止事件时补发 lifecycle end/error（文档声明：`/code/openclaw/docs/concepts/agent-loop.md` 的 `How it works`）。

`runEmbeddedPiAgent` 是 OpenClaw embedded pi runtime 的主入口；导出路径在 `/code/openclaw/src/agents/pi-embedded.ts`，实现位于 `/code/openclaw/src/agents/pi-embedded-runner/run.ts` 的 `export async function runEmbeddedPiAgent`（源码事实：`/code/openclaw/src/agents/pi-embedded.ts`，`/code/openclaw/src/agents/pi-embedded-runner/run.ts`）。

`runEmbeddedPiAgent` 先补齐 `sessionKey`，再解析 `sessionLane = resolveSessionLane(...)` 和 `globalLane = resolveGlobalLane(...)`；默认情况下它先用 `enqueueCommandInLane(sessionLane, ...)` 串行化同一 session，再用 `enqueueCommandInLane(globalLane, ...)` 进入全局并发控制（源码事实：`/code/openclaw/src/agents/pi-embedded-runner/run.ts` 的 `backfillSessionKey`、`runEmbeddedPiAgent`、`enqueueSession`、`enqueueGlobal`）。

OpenClaw 的 stream bridge 是 `subscribeEmbeddedPiSession`：agent loop 文档声明它把 pi-agent-core event 映射成 `stream: "tool"`、`stream: "assistant"`、`stream: "lifecycle"`，代码入口在 `/code/openclaw/src/agents/pi-embedded-subscribe.ts` 的 `subscribeEmbeddedPiSession`，运行尝试在 `/code/openclaw/src/agents/pi-embedded-runner/run/attempt.ts` 中调用该函数（文档声明和源码事实：`/code/openclaw/docs/concepts/agent-loop.md`，`/code/openclaw/src/agents/pi-embedded-subscribe.ts`，`/code/openclaw/src/agents/pi-embedded-runner/run/attempt.ts`）。

tool、assistant、lifecycle 三类事件不是 UI 装饰，而是 agent run 的外部可观测接口：文档列出当前 event streams 为 `lifecycle`、`assistant`、`tool`，并说明 chat channel 在 lifecycle end/error 时发送 final（文档声明：`/code/openclaw/docs/concepts/agent-loop.md` 的 `Event streams (today)` 和 `Chat channel handling`）。

## Hermes 怎么做

Hermes 的主执行对象是 `AIAgent`。`AIAgent.__init__` 接收 provider/model、tool callbacks、platform/user/chat/thread/gateway session 参数、`session_db`、`prefill_messages`、`ephemeral_system_prompt`、`max_iterations` 等运行时配置（源码事实：`/code/hermes-agent/run_agent.py` 的 `class AIAgent` 和 `def __init__`）。

Hermes 在初始化阶段设置 session logging：如果外部没有传入 `session_id`，它生成时间戳加短 uuid 的 `session_id`，并把 JSON session log 放到 `~/.hermes/sessions/session_<session_id>.json`（源码事实：`/code/hermes-agent/run_agent.py` 的 `self.session_start`、`self.session_id`、`self.session_log_file`）。

Hermes 的 system prompt 由 `_build_system_prompt` 分层构造：identity、Hermes help guidance、memory/session/skill/kanban guidance、tool-use enforcement、built-in memory、external `MemoryManager` prompt、skills prompt、context files、timestamp/model/provider 等信息会进入 prompt；注释说明该 prompt 每个 session 缓存，只在 context compression 后重建（源码事实：`/code/hermes-agent/run_agent.py` 的 `_build_system_prompt`）。

`run_conversation` 是 Hermes 的主 loop。它先确保 DB session 存在，复制 `conversation_history` 到本地 `messages`，追加当前 `{"role": "user", "content": user_message}`，然后构建或复用缓存的 system prompt（源码事实：`/code/hermes-agent/run_agent.py` 的 `run_conversation`）。

Hermes 在进入主 loop 前做 preflight context compression：当估算 token 超过 `context_compressor.threshold_tokens` 时调用 `_compress_context`，并在 compression 生成新 session 时把 `conversation_history` 清为 `None`，避免后续 DB flush 错过压缩后的消息（源码事实：`/code/hermes-agent/run_agent.py` 的 `Preflight context compression` 分支）。

Hermes 的 API 请求消息不是直接持久化的 `messages`：每轮会复制出 `api_messages`，把 external memory prefetch 和 plugin context 注入当前用户消息，把 `ephemeral_system_prompt` 拼到 API-call-time system prompt，并插入 `prefill_messages`；注释明确这些注入不写入 session DB（源码事实：`/code/hermes-agent/run_agent.py` 的 `Prepare messages for API call`、`effective_system`、`prefill_messages`）。

Hermes 的 tool calls 通过 `_execute_tool_calls` 分发。该方法会根据 `_should_parallelize_tool_batch` 选择 sequential 或 concurrent 路径；每个 tool result 以 `{"role": "tool", "name": ..., "content": ..., "tool_call_id": ...}` 追加回 `messages`，然后下一轮模型调用会看到这些结果（源码事实：`/code/hermes-agent/run_agent.py` 的 `_execute_tool_calls`、`_execute_tool_calls_sequential`、`_execute_tool_calls_concurrent`）。

Hermes 的 tool routing 分多层：内建 `todo`、`session_search`、`memory`、`clarify`、`delegate_task` 在 `AIAgent` 内部分支处理；external memory provider tool 交给 `self._memory_manager.handle_tool_call`；其他工具走 `handle_function_call`（源码事实：`/code/hermes-agent/run_agent.py` 的 `_invoke_tool` 和 `_execute_tool_calls_sequential`）。

Hermes 的 final response 在没有 tool calls 的分支产生：`assistant_message.content` 被清理 thinking block 后作为 `final_response`，最终追加 assistant message；如果达到 iteration limit，则 `_handle_max_iterations` 会追加一个 summary request 并发起一次不带 tools 的总结请求（源码事实：`/code/hermes-agent/run_agent.py` 的 no-tool-call 分支和 `_handle_max_iterations`）。

Hermes 在 loop 结束后调用 `_persist_session(messages, conversation_history)`，该函数写 JSON session log，并通过 `_flush_messages_to_session_db` 把未写过的新消息追加到 SQLite session store；返回值包含 `final_response`、`last_reasoning`、`messages`、`api_calls`、`completed`、`turn_exit_reason`、`interrupted` 等字段（源码事实：`/code/hermes-agent/run_agent.py` 的 `_persist_session`、`_flush_messages_to_session_db`、`run_conversation` 返回结果构造）。

## 异同点表

| 维度 | OpenClaw | Hermes | 学习结论 |
|---|---|---|---|
| 入口形态 | Gateway RPC `agent` 立即 ack，后台 `agentCommand` 执行，`agent.wait` 等待 lifecycle 终态（`/code/openclaw/docs/concepts/agent-loop.md`）。 | 直接调用 `AIAgent.run_conversation(...)` 返回完整 dict；gateway/CLI 可传入 `conversation_history` 和 callbacks（`/code/hermes-agent/run_agent.py`）。 | OpenClaw 把“接受 run”和“等待结果”拆开；Hermes 的核心抽象是一次同步 conversation turn。 |
| 运行核心 | `runEmbeddedPiAgent` 包装 pi-agent-core runtime，并通过 session/global lane 串行化（`/code/openclaw/src/agents/pi-embedded-runner/run.ts`）。 | `AIAgent.run_conversation` 在 Python 中维护 `messages`、API 调用、tool execution、compression、final response（`/code/hermes-agent/run_agent.py`）。 | OpenClaw 更像 runtime adapter；Hermes 更像单体主循环。 |
| prompt 组装 | 文档声明系统 prompt 来自 base prompt、skills prompt、bootstrap context、per-run overrides（`/code/openclaw/docs/concepts/agent-loop.md`）。 | `_build_system_prompt` 明确按 identity、guidance、memory、skills、context files、timestamp 等层组装并缓存（`/code/hermes-agent/run_agent.py`）。 | 两者都把长期上下文放进 prompt，但 Hermes 的层次在 `run_agent.py` 中更集中可读。 |
| tool result 回流 | stream bridge 把 tool events 暴露为 `stream: "tool"`；最终 payload 也会处理 tool summaries 和 messaging tool 去重（`/code/openclaw/docs/concepts/agent-loop.md`，`/code/openclaw/src/agents/pi-embedded-subscribe.ts`）。 | 每个工具执行后追加 role=`tool` 的 message，带 `tool_call_id`，下一轮 API 调用消费它（`/code/hermes-agent/run_agent.py`）。 | OpenClaw 强调事件流可观测性；Hermes 强调 OpenAI-style message 序列的闭环。 |
| streaming | assistant/tool/lifecycle stream 是正式 agent event stream（`/code/openclaw/docs/concepts/agent-loop.md`）。 | streaming 走 `_interruptible_streaming_api_call`，并通过 callbacks/stream_delta 交给调用方（`/code/hermes-agent/run_agent.py`）。 | OpenClaw 的 stream 是 run 级 event bus；Hermes 的 stream 是 `AIAgent` 内部 API 调用和 UI callback 的一部分。 |
| context compression | 文档声明 auto-compaction 会发 `compaction` stream events 并可能 retry（`/code/openclaw/docs/concepts/agent-loop.md`）。 | `run_conversation` preflight 和 tool loop 后都会调用 `_compress_context`，compression 后重置 `conversation_history`（`/code/hermes-agent/run_agent.py`）。 | 两者都有压缩，但 OpenClaw 文档以事件和 retry 描述，Hermes 代码以 message 列表重写和 DB flush 边界体现。 |

## 源码阅读路线

1. 先读 OpenClaw 生命周期说明：`/code/openclaw/docs/concepts/agent-loop.md`，重点看 `How it works`、`Queueing + concurrency`、`Event streams (today)`。
2. 再读 OpenClaw 导出与主入口：`/code/openclaw/src/agents/pi-embedded.ts`、`/code/openclaw/src/agents/pi-embedded-runner/run.ts` 的 `runEmbeddedPiAgent`。
3. 继续读 OpenClaw stream bridge：`/code/openclaw/src/agents/pi-embedded-subscribe.ts` 的 `subscribeEmbeddedPiSession`，以及 `/code/openclaw/src/agents/pi-embedded-runner/run/attempt.ts` 中调用它的位置。
4. 读 Hermes agent 初始化：`/code/hermes-agent/run_agent.py` 的 `class AIAgent`、`__init__`，重点看 session logging、tool schema、`MemoryManager`、`ContextCompressor` 初始化。
5. 读 Hermes prompt：`/code/hermes-agent/run_agent.py` 的 `_build_system_prompt`。
6. 读 Hermes 主循环：`/code/hermes-agent/run_agent.py` 的 `run_conversation`，从 `messages = list(conversation_history)` 读到返回 result。
7. 读 Hermes tool 执行：`/code/hermes-agent/run_agent.py` 的 `_execute_tool_calls`、`_invoke_tool`、`_execute_tool_calls_sequential`、`_execute_tool_calls_concurrent`。

## 最小复现抽象

以下是学习抽象，不是 OpenClaw 或 Hermes 的精确实现。

- `AgentRun`：持有 `run_id`、`session_id`、输入消息、状态、取消信号和最终结果；对应 OpenClaw 的 `runId`/lifecycle 概念（`/code/openclaw/docs/concepts/agent-loop.md`）和 Hermes 的 `run_conversation` result（`/code/hermes-agent/run_agent.py`）。
- `PromptBuilder`：把系统身份、session context、memory、skills、context files、per-turn injection 组装成 API 可用 prompt；参考 Hermes `_build_system_prompt` 和 OpenClaw agent-loop 文档的 prompt assembly 描述。
- `ModelClient`：封装 provider/model、streaming/non-streaming API 调用、retry 和 usage 读取；参考 Hermes `_build_api_kwargs`/API call loop 与 OpenClaw `resolveModelAsync` 后进入 embedded backend 的路径（`/code/hermes-agent/run_agent.py`，`/code/openclaw/src/agents/pi-embedded-runner/run.ts`）。
- `ToolExecutor`：接收 tool calls，执行工具，生成 role=`tool` 或 stream tool event；参考 Hermes `_execute_tool_calls` 和 OpenClaw `subscribeEmbeddedPiSession` 的 tool stream。
- `StreamEmitter`：统一发送 assistant delta、tool progress、lifecycle start/end/error、compaction events；参考 OpenClaw `stream: "assistant"|"tool"|"lifecycle"` 和 Hermes callbacks。
- `RunFinalizer`：清理临时 scaffolding、写 transcript/session DB、计算 usage、返回 final response；参考 Hermes `_persist_session` 和 OpenClaw agent-loop 文档的 persistence/final payload 描述。

## 容易误解的点

- 不要把 OpenClaw 的 `agent` RPC ack 当成最终回复；文档明确 `agent` 立即返回 `{ runId, acceptedAt }`，最终状态要靠 stream 或 `agent.wait`（`/code/openclaw/docs/concepts/agent-loop.md`）。
- 不要把 Hermes 的 `api_messages` 当成持久化 transcript；代码注释说明 memory/plugin/prefill/ephemeral system prompt 注入是 API-call-time only，原始 `messages` 不被这些注入污染（`/code/hermes-agent/run_agent.py`）。
- 不要把 tool event 和 tool result 混为一谈；OpenClaw 暴露 tool stream event，Hermes 把 tool result 写入 message list 的 role=`tool` 项，两者的外形不同（`/code/openclaw/src/agents/pi-embedded-subscribe.ts`，`/code/hermes-agent/run_agent.py`）。
- 不要假设 compression 只是“摘要一下文本”；Hermes compression 会影响 session 写入边界，代码在 compression 后把 `conversation_history` 置空以保证压缩后的消息完整写入新 session（`/code/hermes-agent/run_agent.py`）。

## 待核实问题

- OpenClaw 文档提到 `agentCommand`，本章已依据 `/code/openclaw/docs/concepts/agent-loop.md` 写入；还需要进一步定位当前 commit 中 `agentCommand` 的精确源码文件和函数实现。
- OpenClaw 的 `tool-assistant-lifecycle` 要求在本章按 `tool`、`assistant`、`lifecycle` 三类 stream 事件写入；本次在 `/code/openclaw/src` 和 `/code/openclaw/docs` 中未找到精确名为 `tool-assistant-lifecycle` 的符号，后续如需更细粒度可继续读 `/code/openclaw/src/gateway/server-chat.agent-events.test.ts` 和 `/code/openclaw/src/agents/pi-embedded-subscribe.handlers.lifecycle.ts`。
- Hermes streaming callbacks 与 gateway adapter 的具体投递链路本章只写到 `AIAgent` 内部 callbacks；gateway 外层如何把 callback 转成各平台消息留待 Gateway 章节核实。
