# Session 与队列

> 参考版本：
> - OpenClaw：`/code/openclaw@ca31a705d02e42ffcfb2c5884bb55339a6d0cbdc`
> - Hermes Agent：`/code/hermes-agent@3d4297a59a8607ed24850524d229f5f42520d087`
> - Codex：OpenAI Codex manual，核验于 2026-06-13：`https://developers.openai.com/codex/codex-manual.md`
> - Claude Code：Anthropic Agent SDK sessions 文档，核验于 2026-06-13：`https://code.claude.com/docs/en/agent-sdk/sessions`

## 本章问题

长生命周期 agent 会同时面对多入口、多消息、多后台任务和长时间工具执行。Session queue 要解决两个问题：同一个 session 内不要并发改同一份 transcript；不同 session 和后台 lane 又要能在可控上限内并行。OpenClaw 的 queue 文档明确说 inbound auto-reply runs 通过 lane-aware FIFO queue 串行化，以避免多个 agent runs 互相碰撞（文档声明：`/code/openclaw/docs/concepts/queue.md`）。

Hermes 的 session 机制主要解决“消息来自哪里、该接到哪个历史、结果应投递到哪里、历史如何恢复”的问题。`gateway/session.py` 的模块说明列出 Session context tracking、Session storage、Reset policy evaluation、Dynamic system prompt injection（源码事实：`/code/hermes-agent/gateway/session.py`）。

## OpenClaw 怎么做

OpenClaw 的 session 身份以 `sessionKey/sessionId` 贯穿 agent loop。agent-loop 文档说明 `agent` RPC 会解析 `sessionKey/sessionId` 并持久化 session metadata；`runEmbeddedPiAgent` 会在早期用 `backfillSessionKey` 从 `sessionId` 回填 `sessionKey`，使 hooks、LCM、compaction 等下游拿到非空 key（文档声明和源码事实：`/code/openclaw/docs/concepts/agent-loop.md`，`/code/openclaw/src/agents/pi-embedded-runner/run.ts` 的 `backfillSessionKey`）。

OpenClaw 的 per-session lane 来自 `resolveSessionLane(params.sessionKey?.trim() || params.sessionId)`；文档说明 lane 名形如 `session:<key>`，保证同一 session 只有一个 active run（文档声明和源码事实：`/code/openclaw/docs/concepts/queue.md`，`/code/openclaw/src/agents/pi-embedded-runner/run.ts` 的 `runEmbeddedPiAgent`）。

OpenClaw 的 global lane 是第二层并发闸门。`runEmbeddedPiAgent` 解析 `globalLane = resolveGlobalLane(params.lane)`，再把 session lane 里的实际任务送入 global lane；queue 文档说明默认 global lane 是 `main`，由 `agents.defaults.maxConcurrent` 控制总体并行度（文档声明和源码事实：`/code/openclaw/docs/concepts/queue.md`，`/code/openclaw/src/agents/pi-embedded-runner/run.ts`）。

`/code/openclaw/src/process/command-queue.ts` 实现了 lane-aware FIFO：`LaneState` 保存 `queue`、`activeTaskIds`、`maxConcurrent`、`draining`、`generation`，`enqueueCommandInLane` 把任务压入指定 lane，`drainLane` 在 `activeTaskIds.size < maxConcurrent` 时取出执行（源码事实：`/code/openclaw/src/process/command-queue.ts`）。

OpenClaw 的 queue modes 是 inbound 消息进入 session lane 前的行为选择。文档列出 `steer`、legacy `queue`、`followup`、`collect`、`steer-backlog`、legacy `interrupt`；默认配置为 `mode: "steer"`、`debounceMs: 500`、`cap: 20`、`drop: "summarize"`（文档声明：`/code/openclaw/docs/concepts/queue.md`）。

OpenClaw 的 `steer` 语义是把 steering messages 送入 active runtime，并在当前 assistant turn 完成 tool calls 后、下一次 LLM call 前交付；如果 run 不能接收 steering，则 fallback 到 followup queue entry（文档声明：`/code/openclaw/docs/concepts/queue.md`）。

OpenClaw 还用 transcript write lock 保护 session file。agent-loop 文档说明 transcript writes 受 session write lock 保护，锁是 process-aware、file-based，写入者最多等待 `session.writeLock.acquireTimeoutMs`，默认 `60000` ms；`acquireSessionWriteLock` 的实现接收 `sessionFile`、`timeoutMs`、`allowReentrant`，并使用 lock file payload 记录 pid 和 createdAt（文档声明和源码事实：`/code/openclaw/docs/concepts/agent-loop.md`，`/code/openclaw/src/agents/session-write-lock.ts`）。

OpenClaw 的 session store 本身也有写队列。`saveSessionStore`、`updateSessionStore` 和 `updateSessionStoreEntry` 都通过 `runExclusiveSessionStoreWrite` 包住写入；`recordSessionMetaFromInbound` 会从 inbound context 派生 metadata 并更新 session store（源码事实：`/code/openclaw/src/config/sessions/store.ts`）。

## Hermes 怎么做

Hermes 的 gateway session 身份从 `SessionSource` 开始。`SessionSource` 描述 platform、chat_id、chat_type、user_id、thread_id、guild_id、parent_chat_id、message_id 等来源字段；注释说明这些信息用于 response routing、system prompt context、cron job delivery origin（源码事实：`/code/hermes-agent/gateway/session.py` 的 `SessionSource`）。

Hermes 的 `SessionContext` 包含 `source`、`connected_platforms`、`home_channels`、`shared_multi_user_session`、`session_key`、`session_id`、created/updated 时间；`build_session_context_prompt` 会把这些信息转成动态 system prompt，说明消息来源、可用平台和 delivery target 约束（源码事实：`/code/hermes-agent/gateway/session.py` 的 `SessionContext` 和 `build_session_context_prompt`）。

Hermes 的 deterministic session key 由 `build_session_key` 生成。代码注释列出 DM、group/channel、thread、per-user isolation 的规则；例如 DM 有 chat_id 时使用 `agent:main:<platform>:dm:<chat_id>`，thread 默认在参与者之间共享，除非显式启用 `thread_sessions_per_user`（源码事实：`/code/hermes-agent/gateway/session.py` 的 `build_session_key`）。

Hermes 的 `SessionStore` 使用 SQLite 作为 session metadata 和 transcript 的主存储，并在不可用时 fallback 到 legacy JSONL；同时 `sessions.json` 仍保存 session key 到 session id 的索引映射（源码事实：`/code/hermes-agent/gateway/session.py` 的 `SessionStore`、`_save`、`append_to_transcript`）。

Hermes 的 `get_or_create_session` 在持有 `self._lock` 时加载和更新 `_entries`，根据 reset policy、`suspended`、`resume_pending` 决定复用或创建新的 `session_id`；SQLite 的 `end_session`/`create_session` 在锁外执行，避免持锁做 I/O（源码事实：`/code/hermes-agent/gateway/session.py` 的 `get_or_create_session`）。

Hermes 的 conversation persistence 发生在 `AIAgent` 内。`run_conversation` 复制 `conversation_history`，追加本轮 user message；结束时 `_persist_session` 同时写 JSON log 和 SQLite；`_flush_messages_to_session_db` 用 `_last_flushed_db_idx` 防止重复写入（源码事实：`/code/hermes-agent/run_agent.py` 的 `run_conversation`、`_persist_session`、`_flush_messages_to_session_db`）。

Hermes 的 gateway routing context 会进入 agent 初始化和 prompt。`AIAgent.__init__` 接收 `platform`、`user_id`、`user_name`、`chat_id`、`chat_name`、`chat_type`、`thread_id`、`gateway_session_key`；`MemoryManager` 初始化时也会把这些字段作为 provider scoping 参数传入（源码事实：`/code/hermes-agent/run_agent.py` 的 `AIAgent.__init__`）。

Hermes 已核实的 interrupt 行为在 `AIAgent` 内：`interrupt()` 设置 `_interrupt_requested`，把 thread-scoped interrupt 同步到 agent execution thread 和 tool worker threads；tool loop 在 API call 前、tool batch 前和 tool 执行中检查 `_interrupt_requested` 并跳出或追加 skipped tool result（源码事实：`/code/hermes-agent/run_agent.py` 的 `interrupt`、`clear_interrupt`、`run_conversation`、`_execute_tool_calls_sequential`、`_execute_tool_calls_concurrent`）。

Hermes 已核实的 steering 行为不是 OpenClaw queue mode，而是 `AIAgent.steer(text)` 保存 `_pending_steer`；主 loop 在 pre-API-call 或 tool result 后通过 `_drain_pending_steer`、`_apply_pending_steer_to_tool_results` 把 guidance 注入 tool result，使下一轮模型调用可见（源码事实：`/code/hermes-agent/run_agent.py` 的 `steer`、`_drain_pending_steer`、`_apply_pending_steer_to_tool_results`）。

Hermes 的 restart interruption 状态在 `SessionEntry.resume_pending` 和相关方法中表达：`mark_resume_pending` 会保留原 `session_id` 和 transcript，`get_or_create_session` 在 `resume_pending` 时返回原 entry，`clear_resume_pending` 在成功 resumed turn 后清除标志；`suspended` 则表示下次 access 强制 reset（源码事实：`/code/hermes-agent/gateway/session.py` 的 `SessionEntry`、`mark_resume_pending`、`get_or_create_session`、`clear_resume_pending`、`suspend_session`）。

## Codex / Claude Code 参考

Codex SDK 把连续工作组织成 thread：TypeScript 示例使用 `codex.startThread()` 后 `thread.run(...)`，同一个 thread 可继续 run，也可用 thread ID resume；Python SDK 通过本地 app-server JSON-RPC 控制 Codex，并允许在 thread 创建或后续 turn 上设置 sandbox。这个接口表达的是“session/thread continuity”，不是内置公开的 per-session WebSocket queue（官方文档：`https://developers.openai.com/codex/codex-manual.md` 的 `Codex SDK`、`Codex App Server`）。

Codex app automations 进一步说明后台 run 需要隔离策略：project automation 可以在 local project 或新 worktree 中运行；thread automation 则是附着当前 thread 的 heartbeat-style wake-up。也就是说，在 Codex 产品形态里，排队和隔离不只靠 session id，还包括 worktree、sandbox、approval policy 和 automation 类型（官方文档：`https://developers.openai.com/codex/codex-manual.md` 的 `Automations`、`Worktrees`）。

Claude Agent SDK 的 session 文档把 session 定义为 SDK 累积的 conversation history，包含 prompt、tool calls、tool results 和 responses，并自动写到磁盘；它支持 one-shot、同进程 multi-turn、`continue`、`resume` 和 `fork`，但文档同时说明 session persistence 不等于 filesystem snapshot，文件回滚要用 checkpointing（官方文档：`https://code.claude.com/docs/en/agent-sdk/sessions`）。

所以把 Codex/Claude Code 纳入本章时，结论不是“它们也有 OpenClaw lane”。更稳妥的说法是：CLI/SDK/MCP host 仍需要定义 session/thread、resume/fork、cancel/interrupt、并发上限和写入隔离；OpenClaw 给出显式 lane queue 的样板，Hermes 给出 gateway session store 和 agent 内 interrupt/steer 样板，Codex/Claude Code 则给出 thread/session API 与宿主需要承担的队列边界。

## 异同点表

| 维度 | OpenClaw | Hermes | 学习结论 |
|---|---|---|---|
| session 身份 | `sessionKey/sessionId` 贯穿 RPC、run、hook、compaction；`runEmbeddedPiAgent` 会回填 `sessionKey`（`/code/openclaw/src/agents/pi-embedded-runner/run.ts`）。 | `SessionSource` 生成 deterministic `session_key`，`SessionEntry` 映射到当前 `session_id`（`/code/hermes-agent/gateway/session.py`）。 | 两者都有 stable key 和 concrete session id 的区分，但生成规则和入口形态不同。 |
| per-session 串行化 | `session:<key>` lane 保证同一 session 一个 active run（`/code/openclaw/docs/concepts/queue.md`）。 | 本章未在 Hermes 证据中确认等价的 per-session run queue；已确认 `SessionStore` 用 `threading.Lock` 保护 `_entries`，`AIAgent` 用 message history 维护单 turn loop（`/code/hermes-agent/gateway/session.py`，`/code/hermes-agent/run_agent.py`）。 | OpenClaw 对 run 级并发有显式 lane；Hermes 已核实部分更偏 session 映射和单个 agent loop 内状态。 |
| global 并发 | `globalLane` 默认 `main`，由 `agents.defaults.maxConcurrent` 控制（`/code/openclaw/docs/concepts/queue.md`，`/code/openclaw/src/agents/pi-embedded-runner/run.ts`）。 | 本章未核实 Hermes 是否有全局 run lane；只核实 tool batch 可在 `ThreadPoolExecutor` 中并发执行（`/code/hermes-agent/run_agent.py`）。 | 不把 Hermes tool 并发等同于 gateway run queue。 |
| queue modes | `steer`、`queue`、`followup`、`collect`、`steer-backlog`、`interrupt` 等模式有文档化优先级和默认值（`/code/openclaw/docs/concepts/queue.md`）。 | 已核实 `AIAgent.steer` 是 pending steer 文本注入机制，`interrupt()` 是停止当前 loop/tool 的机制；未核实 gateway 层有同名 queue modes（`/code/hermes-agent/run_agent.py`）。 | OpenClaw 把消息排队策略配置化；Hermes 已核实的是 agent 内 steer/interrupt 原语。 |
| transcript 保护 | session transcript writes 由 process-aware file-based write lock 保护，默认等待 `60000` ms（`/code/openclaw/docs/concepts/agent-loop.md`，`/code/openclaw/src/agents/session-write-lock.ts`）。 | session index 写入用 `self._lock` 和 atomic replace；SQLite flush 用 `_last_flushed_db_idx` 防重复；JSONL append/rewrite 在 `SessionStore` 方法中执行（`/code/hermes-agent/gateway/session.py`，`/code/hermes-agent/run_agent.py`）。 | OpenClaw 对 transcript 文件有专门 file lock；Hermes 已核实的是 Python lock、SQLite/JSONL 持久化和 flush cursor。 |
| routing context | session metadata 与 Gateway/channel 绑定，agent loop 文档强调 session/workspace preparation（`/code/openclaw/docs/concepts/agent-loop.md`）。 | `SessionContext` 和 `build_session_context_prompt` 明确把来源、平台、home channels、delivery target 约束注入 prompt（`/code/hermes-agent/gateway/session.py`）。 | Hermes 的 routing context 在源码中更直接体现在 prompt 构造；OpenClaw 相关细节分散在 Gateway/session/channel 文档。 |

## Codex / Claude Code 参照表

| 维度 | Codex | Claude Code | 学习结论 |
|---|---|---|---|
| session/thread | SDK 使用 thread start/resume/run；app-server 作为本地 JSON-RPC 控制面。 | Agent SDK sessions 保存 conversation history，支持 one-shot、continue、resume、fork。 | session continuity 是 host API 能力，不等于自动存在 run queue。 |
| 并发隔离 | automation 可在 local project 或 worktree 中运行，sandbox/approval 参与隔离。 | scheduled prompt 在当前 turn 后低优先级入队；session task 有上限。 | 同一 session 是否串行、后台任务是否隔离，必须由宿主策略定义。 |
| 持久化边界 | Codex thread/workspace/state 由 CLI/app/SDK host 管理。 | SDK session persistence 保存 conversation，不是 filesystem snapshot。 | transcript、workspace snapshot 和 tool side effects 不能混为一谈。 |
| cancel/interrupt | SDK/CLI/app-server 可作为取消和权限策略承载面。 | sessions、permissions、hooks 和 scheduled tasks 共同影响 turn lifecycle。 | 最小复现要预留 run cancellation 与 policy stop，而不是只做消息 append。 |

## 源码阅读路线

1. 先读 OpenClaw queue 文档：`/code/openclaw/docs/concepts/queue.md`，重点看 `How it works`、`Queue modes`、`Precedence`、`Scope and guarantees`。
2. 读 OpenClaw agent loop 的并发和锁说明：`/code/openclaw/docs/concepts/agent-loop.md` 的 `Queueing + concurrency`、`Session + workspace preparation`。
3. 读 OpenClaw lane 实现：`/code/openclaw/src/agents/pi-embedded-runner/run.ts` 的 `runEmbeddedPiAgent`，确认 `sessionLane` 和 `globalLane` 的入队顺序。
4. 读 OpenClaw queue runtime：`/code/openclaw/src/process/command-queue.ts` 的 `LaneState`、`enqueueCommandInLane`、`drainLane`。
5. 读 OpenClaw session store：`/code/openclaw/src/config/sessions/store.ts` 的 `recordSessionMetaFromInbound`、`updateSessionStore`、`saveSessionStore`。
6. 读 OpenClaw transcript write lock：`/code/openclaw/src/agents/session-write-lock.ts` 的 `resolveSessionWriteLockAcquireTimeoutMs`、`acquireSessionWriteLock`。
7. 读 Hermes gateway session：`/code/hermes-agent/gateway/session.py` 的 `SessionSource`、`SessionContext`、`build_session_key`、`SessionEntry`、`SessionStore`。
8. 读 Hermes persistence：`/code/hermes-agent/run_agent.py` 的 `_persist_session`、`_flush_messages_to_session_db`、`run_conversation`。
9. 读 Hermes interrupt/steer：`/code/hermes-agent/run_agent.py` 的 `interrupt`、`clear_interrupt`、`steer`、`_apply_pending_steer_to_tool_results`。
10. 读 Codex manual 的 `Codex SDK`、`Codex App Server`、`Automations`、`Worktrees`，确认 thread/run、resume、worktree 隔离和后台 automation 的边界。
11. 读 Claude Agent SDK 的 `Work with sessions`，确认 `continue`、`resume`、`fork` 与 session persistence 的语义。

## 最小复现抽象

以下是学习抽象，不是 OpenClaw 或 Hermes 的精确实现。

- `SessionId`：包含 stable key 和 concrete transcript id；参考 OpenClaw `sessionKey/sessionId` 和 Hermes `SessionEntry.session_key/session_id`（`/code/openclaw/docs/concepts/agent-loop.md`，`/code/hermes-agent/gateway/session.py`）。
- `SessionStore`：负责 session key 生成、metadata 保存、transcript 定位、reset/resume 状态；参考 OpenClaw `/code/openclaw/src/config/sessions/store.ts` 和 Hermes `SessionStore`。
- `SessionLane`：同一 session 的 run 串行化单元；参考 OpenClaw `session:<key>` lane（`/code/openclaw/docs/concepts/queue.md`，`/code/openclaw/src/agents/pi-embedded-runner/run.ts`）。
- `RunQueue`：管理 lane FIFO、并发上限、等待告警、任务超时释放；参考 OpenClaw `command-queue.ts`。
- `TranscriptLock`：保护 transcript 文件或 DB 写入，避免跨进程/跨线程并发写损坏；参考 OpenClaw `session-write-lock.ts` 和 Hermes `SessionStore._lock`/SQLite append 路径。
- `InterruptController`：提供 `interrupt`、`clear_interrupt`、`steer` 或等价接口；参考 Hermes `AIAgent.interrupt`/`steer` 和 OpenClaw queue 文档中的 `interrupt`/`steer` mode。
- `ThreadHandle`：封装 Codex/Claude Code 这类 SDK host 的 thread/session id、resume/fork 和 per-turn sandbox/permission 选项；它不替代 `SessionLane`，而是 `RunQueue` 的输入之一。

## 容易误解的点

- 不要把 Hermes 的 `steer()` 直接等同于 OpenClaw 的 `messages.queue.mode = "steer"`；OpenClaw 的 steer 是 gateway queue mode，Hermes 已核实的是 `AIAgent` 内部 pending steer 注入机制（`/code/openclaw/docs/concepts/queue.md`，`/code/hermes-agent/run_agent.py`）。
- 不要把 OpenClaw 的 per-session lane 当成唯一锁；文档明确 transcript writes 还有 process-aware file-based session write lock，用于捕获绕过 in-process queue 或来自其他进程的写入（`/code/openclaw/docs/concepts/agent-loop.md`）。
- 不要把 Hermes 的 `SessionStore._lock` 理解为 agent run 队列；该锁在本章证据中保护 `_entries` 和 JSONL append/rewrite，未证明它串行化同一 session 的所有 agent runs（`/code/hermes-agent/gateway/session.py`）。
- 不要把 `resume_pending` 写成“自动成功恢复”；源码只证明它保留原 `session_id` 并在下一次 `get_or_create_session` 返回原 entry，成功后由 `clear_resume_pending` 清除（`/code/hermes-agent/gateway/session.py`）。
- 不要把 SDK session persistence 当成文件系统快照；Claude Agent SDK 文档明确 session 保存 conversation history，文件变更回滚属于 checkpointing 问题。
- 不要因为 Codex/Claude Code 有 `resume`/`continue` 就省略队列设计；多入口系统仍要决定同一 thread 是否允许并发 run、取消请求怎样传播、后台 automation 是否隔离到 worktree。

## 待核实问题

- Hermes gateway 外层是否有显式 per-session run queue、全局 run lane 或 active process registry 的完整调度策略，本章只核实了 `SessionStore`、`AIAgent` 内部 steer/interrupt 和 tool 并发。
- Hermes 的 redirect 行为本章未写成事实；`run_agent.py` 中出现“redirect”字样多处属于注释或输出重定向上下文，还需要定位 gateway/message handler 是否有用户消息 redirect 语义。
- OpenClaw queue mode 从 inbound channel 到 `queueEmbeddedPiMessage`/followup entry 的完整调用链尚未展开，本章依据 `/code/openclaw/docs/concepts/queue.md` 和 `runEmbeddedPiAgent` 的 lane 入队写入。
