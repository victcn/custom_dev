# 记忆机制

> 参考版本：
> - OpenClaw：`/code/openclaw@ca31a705d02e42ffcfb2c5884bb55339a6d0cbdc`
> - Hermes Agent：`/code/hermes-agent@3d4297a59a8607ed24850524d229f5f42520d087`

## 本章问题

长生命周期 agent 不能只依赖当前上下文窗口：对话会被压缩，session 会切换，用户偏好、环境事实、工具经验和跨会话召回都需要落到可恢复的存储或 provider 接口里。[学习抽象；锚点：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/design.md` 的 `05-memory.md` 章节边界]

本章把记忆拆成五个问题：写到哪里、怎么召回、怎么注入模型上下文、什么时候写回、谁做后台维护。OpenClaw 的公开文档把基础记忆明确写成 workspace 内 Markdown 文件；Hermes 的源码把外部记忆抽成 `MemoryProvider`，由 `MemoryManager` 编排。[文档声明/源码事实；锚点：`/code/openclaw/docs/concepts/memory.md:9`、`/code/hermes-agent/agent/memory_provider.py:1`、`/code/hermes-agent/agent/memory_manager.py:190`]

## OpenClaw 怎么做

OpenClaw 文档声明，模型只“记得”写入磁盘的内容，没有隐藏状态；记忆文件位于 agent workspace，默认路径是 `~/.openclaw/workspace`。[文档声明；锚点：`/code/openclaw/docs/concepts/memory.md:9`、`/code/openclaw/docs/concepts/memory.md:24`]

OpenClaw 的基础记忆文件有三类：`MEMORY.md` 保存长期事实、偏好和决策，`memory/YYYY-MM-DD.md` 保存日记式运行上下文，`DREAMS.md` 保存 dreaming sweep 和 human review 摘要。[文档声明；锚点：`/code/openclaw/docs/concepts/memory.md:15`]

OpenClaw 的 memory tools 是 `memory_search` 和 `memory_get`：前者做语义/混合检索，后者读取指定记忆文件或行范围；这两个工具由 active memory plugin 提供，默认是 `memory-core`。[文档声明；锚点：`/code/openclaw/docs/concepts/memory.md:42`、`/code/openclaw/docs/concepts/memory.md:50`]

OpenClaw 文档列出多种 memory backends：Builtin 是 SQLite-based，QMD 是 local-first sidecar，Honcho 提供 user modeling/semantic search/multi-agent awareness，LanceDB 是 bundled LanceDB-backed memory。[文档声明；锚点：`/code/openclaw/docs/concepts/memory.md:88`]

OpenClaw 的 memory plugin 能注册 prompt builder、flush plan、runtime 和 public artifacts；`MemoryRuntime` 暴露 `getMemorySearchManager`、`resolveMemoryBackendConfig` 和 `closeAllMemorySearchManagers`。[源码事实；锚点：`/code/openclaw/src/plugins/memory-state.ts:97`、`/code/openclaw/src/plugins/memory-state.ts:128`]

OpenClaw 的 memory host SDK 存储层重新导出 `listMemoryFiles`、`readMemoryFile`、`resolveMemoryBackendConfig`、`MemorySearchManager` 等接口，说明默认 memory engine surface 包含文件枚举、文件读取、backend 解析和搜索管理器类型。[源码事实；锚点：`/code/openclaw/packages/memory-host-sdk/src/engine-storage.ts:3`]

OpenClaw 在 compaction 前有自动 memory flush：文档说 compaction 总结对话前会跑一个 silent turn，提醒 agent 把重要上下文写入 memory files，默认开启；更细的 reference 说明它使用 pre-threshold flush、`NO_REPLY`/`no_reply` 抑制用户可见输出，并在 `agents.defaults.compaction.memoryFlush` 下配置。[文档声明；锚点：`/code/openclaw/docs/concepts/memory.md:118`、`/code/openclaw/docs/reference/session-management-compaction.md:403`、`/code/openclaw/docs/reference/session-management-compaction.md:417`]

OpenClaw 的 memory flush 写入有 append-only guard 测试：`buildEmbeddedAttemptToolRunContext` 会携带 `trigger: "memory"` 和 `memoryFlushWritePath`，测试中的 `wrapToolMemoryFlushAppendOnlyWrite` 只允许向指定 memory path 追加内容。[源码事实；锚点：`/code/openclaw/src/agents/pi-embedded-runner/run/attempt.memory-flush-forwarding.test.ts:38`、`/code/openclaw/src/agents/pi-embedded-runner/run/attempt.memory-flush-forwarding.test.ts:64`]

OpenClaw 的 dreaming 是可选后台 consolidation pass，默认 disabled；它收集 short-term signals，打分候选项，只把合格项提升到 `MEMORY.md`，并把 phase summaries 和 diary entries 写入 `DREAMS.md`。[文档声明；锚点：`/code/openclaw/docs/concepts/memory.md:150`、`/code/openclaw/docs/concepts/dreaming.md:11`]

OpenClaw dreaming 有 Light、Deep、REM 三个 phase：Light 分拣/暂存，Deep 评分并写 `MEMORY.md`，REM 做主题反思；机器状态写在 `memory/.dreams/`，人类可读输出写入 `DREAMS.md` 或 `memory/dreaming/<phase>/YYYY-MM-DD.md`。[文档声明；锚点：`/code/openclaw/docs/concepts/dreaming.md:17`、`/code/openclaw/docs/concepts/dreaming.md:26`]

## Hermes 怎么做

Hermes 的 `MemoryProvider` 是抽象基类，声明 provider 给 agent 提供跨 session persistent recall；其生命周期包括 `initialize()`、`system_prompt_block()`、`prefetch()`、`sync_turn()`、`get_tool_schemas()`、`handle_tool_call()` 和 `shutdown()`。[源码事实；锚点：`/code/hermes-agent/agent/memory_provider.py:1`、`/code/hermes-agent/agent/memory_provider.py:15`]

Hermes 的 `MemoryManager` 是 provider 编排器；源码注释和 `add_provider()` 实现都说明 builtin provider 总是允许，但同时最多只允许一个 external provider，避免 tool schema bloat 和冲突 backend。[源码事实；锚点：`/code/hermes-agent/agent/memory_manager.py:1`、`/code/hermes-agent/agent/memory_manager.py:204`]

Hermes `run_agent.py` 根据 `memory.provider` 选择外部 provider，经 `plugins.memory.load_memory_provider` 加载后加入 `MemoryManager`，并把 `session_id`、`platform`、`hermes_home`、gateway 用户/chat/thread 信息和 profile identity 传给 `initialize_all()`。[源码事实；锚点：`/code/hermes-agent/run_agent.py:1909`、`/code/hermes-agent/run_agent.py:1924`、`/code/hermes-agent/agent/memory_manager.py:538`]

Hermes 会把 provider 的 tool schemas 注入 tool surface；如果同名工具已经存在，会跳过以避免重复函数名。[源码事实；锚点：`/code/hermes-agent/run_agent.py:1972`、`/code/hermes-agent/agent/memory_manager.py:330`]

Hermes 的 `MemoryManager.prefetch_all()` 会遍历所有 provider 召回上下文，失败不阻塞其他 provider；`sync_all()` 在完成 turn 后把 user/assistant 内容同步给所有 provider；`queue_prefetch_all()` 为下一 turn 排队后台召回。[源码事实；锚点：`/code/hermes-agent/agent/memory_manager.py:285`、`/code/hermes-agent/agent/memory_manager.py:304`、`/code/hermes-agent/agent/memory_manager.py:317`]

Hermes 通过 `build_memory_context_block()` 把 prefetched memory 包进 `<memory-context>`，并带 system note，强调这是 recalled memory context，不是新的 user input；`sanitize_context()` 和 `StreamingContextScrubber` 用于剥离或防止 memory-context 泄露到 UI。[源码事实；锚点：`/code/hermes-agent/agent/memory_manager.py:54`、`/code/hermes-agent/agent/memory_manager.py:62`、`/code/hermes-agent/agent/memory_manager.py:173`]

Hermes 的主 loop 在每个用户 turn 开始前调用 `on_turn_start()`，随后只执行一次 `prefetch_all()` 并缓存，避免工具循环里重复调用；构造 API messages 时只把 fenced memory 注入当前 user message，不修改原始 `messages`，因此不会写入 session persistence。[源码事实；锚点：`/code/hermes-agent/run_agent.py:11812`、`/code/hermes-agent/run_agent.py:11822`、`/code/hermes-agent/run_agent.py:11983`]

Hermes 的 memory tool routing 由 `MemoryManager.handle_tool_call()` 通过 `_tool_to_provider` 转发给对应 provider；`run_agent.py` 在内置 `memory` tool 写入 `add`/`replace` 时调用 `on_memory_write()`，让 external provider 镜像内置 memory 写入。[源码事实；锚点：`/code/hermes-agent/agent/memory_manager.py:356`、`/code/hermes-agent/run_agent.py:10301`、`/code/hermes-agent/agent/memory_manager.py:483`]

Hermes 的 `MEMORY_GUIDANCE` 被 `run_agent.py` 在 `memory` tool 可用时注入系统 prompt；它要求 memory 保存 durable facts/user preferences/environment details/tool quirks/stable conventions，并明确把 procedures/workflows 放到 skills 而不是 memory。[源码事实；锚点：`/code/hermes-agent/agent/prompt_builder.py:150`、`/code/hermes-agent/run_agent.py:5647`]

## 异同点表

| 维度 | OpenClaw | Hermes | 学习结论 |
| --- | --- | --- | --- |
| 基础存储形态 | Workspace Markdown 文件：`MEMORY.md`、`memory/YYYY-MM-DD.md`、`DREAMS.md`。[锚点：`/code/openclaw/docs/concepts/memory.md:15`] | 内置 memory store 加可选 external `MemoryProvider`；provider contract 由 `MemoryProvider` 定义。[锚点：`/code/hermes-agent/agent/memory_provider.py:42`] | 记忆可以先以文件落地，也可以先以 provider 接口落地；关键是把存储边界和 prompt 注入边界分开。 |
| 召回工具 | `memory_search`、`memory_get` 由 active memory plugin 提供。[锚点：`/code/openclaw/docs/concepts/memory.md:42`] | Provider 可通过 `get_tool_schemas()` 暴露工具，并由 `MemoryManager.handle_tool_call()` 路由。[锚点：`/code/hermes-agent/agent/memory_manager.py:330`、`/code/hermes-agent/agent/memory_manager.py:356`] | 召回应作为工具或统一接口暴露，而不是把所有记忆直接塞进 prompt。 |
| Backend/Provider 边界 | memory plugin capability 挂载 prompt、flush plan、runtime 和 artifacts。[锚点：`/code/openclaw/src/plugins/memory-state.ts:128`] | `MemoryManager` 管 provider，最多一个 external provider。[锚点：`/code/hermes-agent/agent/memory_manager.py:204`] | 长期系统需要单一编排点，避免多个后端同时污染上下文和工具 schema。 |
| 上下文注入 | 文档强调启动/近期文件和 memory tools；compaction 前通过 silent turn 促使写盘。[锚点：`/code/openclaw/docs/concepts/memory.md:118`] | `prefetch_all()` 的结果被 `<memory-context>` 包裹后注入当前 user message。[锚点：`/code/hermes-agent/run_agent.py:11983`] | 召回内容必须被标记为“背景记忆”，不能被模型误读为新的用户输入。 |
| 压缩前处理 | `agents.defaults.compaction.memoryFlush` 默认开启，pre-threshold silent flush。[锚点：`/code/openclaw/docs/reference/session-management-compaction.md:403`] | Provider contract 有 `on_pre_compress()`，`run_agent.py` 在 compression 前调用 memory manager。[锚点：`/code/hermes-agent/agent/memory_provider.py:202`、`/code/hermes-agent/run_agent.py:10016`] | compaction 前应有记忆写回钩子，否则长期事实容易只留在即将被摘要的上下文里。 |
| 后台维护 | Dreaming 可选，Light/REM/Deep 阶段管理短期信号和长期提升。[锚点：`/code/openclaw/docs/concepts/dreaming.md:26`] | 本章只核实到 provider prefetch/sync/session hooks；未核实到等价 dreaming 后台提升机制。[锚点：`/code/hermes-agent/agent/memory_provider.py:142`] | 后台维护是高级能力；第一版实现可先保留接口，再决定是否做自动提升。 |

## 源码阅读路线

1. OpenClaw 概念入口：`/code/openclaw/docs/concepts/memory.md`，先读 file layout、memory tools、backend、automatic flush、dreaming。[锚点：`/code/openclaw/docs/concepts/memory.md:13`]
2. OpenClaw dreaming 细节：`/code/openclaw/docs/concepts/dreaming.md`，重点读 What dreaming writes、Phase model、Dream Diary、Scheduling。[锚点：`/code/openclaw/docs/concepts/dreaming.md:17`、`/code/openclaw/docs/concepts/dreaming.md:73`、`/code/openclaw/docs/concepts/dreaming.md:110`]
3. OpenClaw compaction flush：`/code/openclaw/docs/reference/session-management-compaction.md` 的 `Pre-compaction "memory flush"`。[锚点：`/code/openclaw/docs/reference/session-management-compaction.md:403`]
4. OpenClaw memory plugin state：`/code/openclaw/src/plugins/memory-state.ts`，看 `MemoryPluginCapability`、`registerMemoryCapability()`、`resolveMemoryFlushPlan()`、`getMemoryRuntime()`。[锚点：`/code/openclaw/src/plugins/memory-state.ts:128`、`/code/openclaw/src/plugins/memory-state.ts:164`、`/code/openclaw/src/plugins/memory-state.ts:256`]
5. OpenClaw memory engine surface：`/code/openclaw/packages/memory-host-sdk/src/engine.ts` 和 `engine-storage.ts`，确认它们重新导出的搜索/读取/backend helper。[锚点：`/code/openclaw/packages/memory-host-sdk/src/engine-storage.ts:3`]
6. Hermes provider contract：`/code/hermes-agent/agent/memory_provider.py`，按生命周期方法阅读。[锚点：`/code/hermes-agent/agent/memory_provider.py:15`]
7. Hermes manager：`/code/hermes-agent/agent/memory_manager.py`，读 `build_memory_context_block()`、`MemoryManager.add_provider()`、`prefetch_all()`、`sync_all()`、`handle_tool_call()`。[锚点：`/code/hermes-agent/agent/memory_manager.py:173`、`/code/hermes-agent/agent/memory_manager.py:204`、`/code/hermes-agent/agent/memory_manager.py:285`]
8. Hermes agent loop 集成：`/code/hermes-agent/run_agent.py`，读 provider 初始化、tool schema 注入、system prompt、prefetch 注入和 tool routing。[锚点：`/code/hermes-agent/run_agent.py:1909`、`/code/hermes-agent/run_agent.py:1972`、`/code/hermes-agent/run_agent.py:5613`、`/code/hermes-agent/run_agent.py:11822`]

## 最小复现抽象

以下是学习抽象，不是 OpenClaw 或 Hermes 的精确实现。

- `MemoryStore`：负责 durable facts、daily notes、user profile 或 provider-side records 的持久化；在 OpenClaw 可映射到 `MEMORY.md`/`memory/YYYY-MM-DD.md`，在 Hermes 可映射到内置 memory store 或 external provider storage。[学习抽象；锚点：`/code/openclaw/docs/concepts/memory.md:15`、`/code/hermes-agent/agent/memory_provider.py:114`]
- `MemoryRetriever`：提供 `search(query)` 和 `get(path/range)`，可用 hybrid search 或 provider prefetch；OpenClaw 对应 `memory_search`/`memory_get`，Hermes 对应 `MemoryProvider.prefetch()` 和 provider tools。[学习抽象；锚点：`/code/openclaw/docs/concepts/memory.md:42`、`/code/hermes-agent/agent/memory_provider.py:92`]
- `MemoryInjector`：把召回内容变成模型可读但不混同用户输入的上下文块；Hermes 可直接抽象自 `<memory-context>` fencing，OpenClaw 可抽象自 prompt/memory plugin prompt section 和 startup memory injection。[学习抽象；锚点：`/code/hermes-agent/agent/memory_manager.py:173`、`/code/openclaw/src/plugins/memory-state.ts:215`]
- `MemoryWriter`：在 turn 完成、用户显式要求、memory flush 或 tool write 时写回；Hermes 的 `sync_all()` 和 `on_memory_write()`、OpenClaw 的 memory files 和 compaction flush 都提供证据。[学习抽象；锚点：`/code/hermes-agent/agent/memory_manager.py:317`、`/code/hermes-agent/agent/memory_manager.py:483`、`/code/openclaw/docs/reference/session-management-compaction.md:403`]
- `MemoryMaintenanceJob`：定期或触发式整理、提升、归档或回填记忆；OpenClaw 的 dreaming 是明确证据，Hermes 侧本章只抽象 provider hooks，不写成已实现等价 dreaming。[学习抽象；锚点：`/code/openclaw/docs/concepts/dreaming.md:110`、`/code/hermes-agent/agent/memory_provider.py:202`]

## 容易误解的点

- 不要把 OpenClaw 的 `DREAMS.md` 当作长期记忆主文件；文档说长期 promotion 仍写 `MEMORY.md`，`DREAMS.md` 是 Dream Diary/人类 review 输出。[锚点：`/code/openclaw/docs/concepts/dreaming.md:17`、`/code/openclaw/docs/concepts/dreaming.md:73`]
- 不要把 OpenClaw 的 automatic memory flush 理解成 compaction summary 本身；reference 说它是在 compaction 前跑 silent turn，把 durable state 写到磁盘。[锚点：`/code/openclaw/docs/reference/session-management-compaction.md:403`]
- 不要把 Hermes 的 prefetched memory 当成用户新输入；`build_memory_context_block()` 明确写入 system note，`run_agent.py` 也只在 API-call-time 注入，不修改原始 `messages`。[锚点：`/code/hermes-agent/agent/memory_manager.py:173`、`/code/hermes-agent/run_agent.py:11983`]
- 不要把 Hermes 的 memory 和 skills 混用：`MEMORY_GUIDANCE` 明确说 procedures/workflows belong in skills, not memory。[锚点：`/code/hermes-agent/agent/prompt_builder.py:150`]
- 不要假设 Hermes 支持多个 external memory providers 并行；`MemoryManager.add_provider()` 对第二个 external provider 直接 warning 并 return。[锚点：`/code/hermes-agent/agent/memory_manager.py:204`]

## 待核实问题

- OpenClaw 文档中的“Today and yesterday's notes are loaded automatically”与 `docs/reference/token-use.md` 中普通 turn 不注入 daily files 的表述可能有上下文差异，本章未完整追踪 startup/reset/DM session 的实际注入路径。[锚点：`/code/openclaw/docs/concepts/memory.md:19`、`/code/openclaw/docs/reference/token-use.md:22`]
- OpenClaw `memory-core` bundled plugin 的具体 `memory_search`/`memory_get` 注册实现和 dreaming implementation 文件未逐行阅读，本章主要依据 docs、plugin state、SDK surface 和测试。[锚点：`/code/openclaw/docs/concepts/memory.md:42`、`/code/openclaw/src/plugins/memory-state.ts:128`]
- Hermes 内置 memory store 的文件格式、`memory` tool 具体实现和 `session_search` 的 transcript recall 细节未在本章展开。[锚点：`/code/hermes-agent/agent/prompt_builder.py:150`、`/code/hermes-agent/run_agent.py:10301`]
- Hermes 是否存在与 OpenClaw dreaming 等价的自动长期记忆提升机制，本章未找到直接证据，不写成事实。[锚点：`/code/hermes-agent/agent/memory_provider.py:142`]
