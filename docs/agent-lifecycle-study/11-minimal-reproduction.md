# 最小复现抽象

> 参考版本：
> - OpenClaw：`/code/openclaw@ca31a705d02e42ffcfb2c5884bb55339a6d0cbdc`
> - Hermes Agent：`/code/hermes-agent@3d4297a59a8607ed24850524d229f5f42520d087`

> 重要说明：本章是学习抽象，不代表 OpenClaw 或 Hermes Agent 的精确实现。它把前面章节的机制边界和本地已核验源码/文档锚点压缩成一个可复现的学习模型，用来理解“长期运行 agent 至少需要哪些层”。锚点：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/design.md`、`/code/code-dev/agent_dev/docs/agent-lifecycle-study/implementation-plan.md`。

## 本章问题

如果只想学习长生命周期 agent，最容易犯的错误是直接复制某个项目的目录结构。更稳妥的方式是先抽象出三层：第一层让 agent 能长期入口化、会话化、工具化、可记录；第二层加入长期记忆、调度、队列和失败恢复；第三层再靠近 OpenClaw/Hermes 的高级能力，例如插件、skill curator、多 agent 委派、沙箱 workspace 和 provider failover。这个三层划分是学习抽象，依据来自章节边界和已核验锚点。锚点：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/design.md`、`/code/openclaw/docs/concepts/architecture.md`、`/code/openclaw/docs/concepts/agent-loop.md`、`/code/openclaw/docs/concepts/agent.md`、`/code/openclaw/docs/concepts/queue.md`、`/code/openclaw/docs/concepts/memory.md`、`/code/hermes-agent/run_agent.py`、`/code/hermes-agent/agent/memory_manager.py`、`/code/hermes-agent/cron/scheduler.py`、`/code/hermes-agent/providers/README.md`。

## OpenClaw 怎么做

OpenClaw 的最外层是单个 long-lived `Gateway`，它拥有 messaging surfaces，control-plane clients 和 nodes 通过 WebSocket 连接；Gateway 是 sessions、channels、tools、events 的控制面。锚点：`/code/openclaw/docs/concepts/architecture.md`、`/code/openclaw/README.md`。

OpenClaw 的 agent loop 是 intake、context assembly、model inference、tool execution、streaming replies、persistence 的权威路径；一次 run 会通过 lifecycle/assistant/tool stream 暴露进度。锚点：`/code/openclaw/docs/concepts/agent-loop.md`。

OpenClaw 的 agent runtime 使用 workspace 作为工具和上下文的工作目录，bootstrap files 会被注入 system prompt；session transcripts 存在 `~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`。锚点：`/code/openclaw/docs/concepts/agent.md`。

OpenClaw 用 lane-aware FIFO queue 串行化同一 session key 的 run，并通过 global lane 控制总体并发；默认 queue mode 是 `steer`，也支持 `followup`、`collect`、`interrupt` 等模式。锚点：`/code/openclaw/docs/concepts/queue.md`。

OpenClaw 的 memory 以 workspace 中的 Markdown 文件和 memory provider 为核心，包括 `MEMORY.md`、`memory/YYYY-MM-DD.md`、可选 `DREAMS.md`，并提供 `memory_search` 与 `memory_get`。锚点：`/code/openclaw/docs/concepts/memory.md`。

OpenClaw 的高级能力包括 sandboxing、skills、plugins/hooks、model failover 等；本章只把它们抽象为高级层组件，不等同于源码实现。锚点：`/code/openclaw/docs/gateway/sandboxing.md`、`/code/openclaw/docs/concepts/agent-loop.md`、`/code/openclaw/docs/concepts/retry.md`。

## Hermes 怎么做

Hermes 的核心运行对象是 `AIAgent`，`run_agent.py` 描述它支持 tool calling conversation loop、multiple model providers 和 `run_conversation()`。锚点：`/code/hermes-agent/run_agent.py`。

Hermes 的 prompt assembly 在 `agent/prompt_builder.py` 中包含 identity、memory guidance、skills guidance、kanban guidance、context-file scanning；其中 context file scanning 会检测 prompt injection patterns 和 invisible unicode。锚点：`/code/hermes-agent/agent/prompt_builder.py`。

Hermes 的 `MemoryManager` 编排 memory providers，提供 `build_system_prompt()`、`prefetch_all()`、`sync_all()`、`queue_prefetch_all()` 这类集成点，并用 fenced memory context 区分召回记忆和新用户输入。锚点：`/code/hermes-agent/agent/memory_manager.py`。

Hermes 的 cron scheduler 支持 scheduled job、skill loading、assembled prompt scanning、delivery target resolution 和 inactivity timeout；cron job 最终调用 `agent.run_conversation(prompt)`。锚点：`/code/hermes-agent/cron/scheduler.py`。

Hermes 的 provider registry 使用 `ProviderProfile`，下游 auth、model listing、runtime routing、transport kwargs、agent transports 都读取 registry/profile，而不是各自维护 provider 逻辑。锚点：`/code/hermes-agent/providers/README.md`。

Hermes README 声明它支持 persistent memory、skill 自我改进、cron scheduling、subagents/parallel workstreams 和多种 terminal backends；本章只把这些作为高级层学习抽象的证据，不展开实现细节。锚点：`/code/hermes-agent/README.md`、`/code/hermes-agent/README.zh-CN.md`。

## 异同点表

| 维度 | OpenClaw | Hermes | 学习结论 |
|---|---|---|---|
| 长期入口 | long-lived `Gateway` 拥有 messaging surfaces 和 WebSocket 控制面。锚点：`/code/openclaw/docs/concepts/architecture.md` | CLI/gateway/cron 最终围绕 `AIAgent.run_conversation()` 运行。锚点：`/code/hermes-agent/run_agent.py`、`/code/hermes-agent/cron/scheduler.py` | 最小复现需要一个入口层，但入口形态可以是 daemon、CLI 或 scheduler。 |
| 会话与 transcript | session transcript 为 JSONL，路径由 OpenClaw 管理。锚点：`/code/openclaw/docs/concepts/agent.md` | 本章未核实 Hermes session DB/transcript 的完整结构，只核实 `run_conversation()` 会持久化 session。锚点：`/code/hermes-agent/run_agent.py` | 学习模型应保留 `SessionStore` 和 `TranscriptStore` 两个边界。 |
| agent loop | 明确定义 intake → context → model → tools → streaming → persistence。锚点：`/code/openclaw/docs/concepts/agent-loop.md` | `AIAgent` 包含 tool calling loop、API retry、tool call validation 和 persistence。锚点：`/code/hermes-agent/run_agent.py` | `AgentLoop` 是第一层核心，不应和 Gateway、memory、scheduler 混成一个概念。 |
| 记忆 | Markdown memory files + memory provider tools。锚点：`/code/openclaw/docs/concepts/memory.md` | `MemoryManager` 统一 provider、prompt、prefetch、sync。锚点：`/code/hermes-agent/agent/memory_manager.py` | 第二层需要 `MemoryStore` 和 `MemoryRetriever`，具体后端可替换。 |
| 队列/lane | lane-aware FIFO，session lane + global lane。锚点：`/code/openclaw/docs/concepts/queue.md` | README 声明 subagents/parallel workstreams；cron scheduler 使用 worker thread 运行 job。锚点：`/code/hermes-agent/README.md`、`/code/hermes-agent/cron/scheduler.py` | 显式 `Queue/Lane` 是本章的学习抽象要求，不表示已核实 Hermes 有 OpenClaw 等价 lane；没有并发边界时，会话和工具状态更容易被污染。 |
| 调度与投递 | cron 文档声明 Gateway 内置 scheduler，heartbeat 文档说明周期性主会话 turn；commitments 源码提供 inferred follow-up store。锚点：`/code/openclaw/docs/automation/cron-jobs.md`、`/code/openclaw/docs/gateway/heartbeat.md`、`/code/openclaw/src/commitments/store.ts` | cron scheduler 解析 delivery targets，组装 prompt 后运行 agent。锚点：`/code/hermes-agent/cron/scheduler.py` | 第二层最小复现应包含 `Scheduler` 和 `DeliveryTarget`。 |
| 插件/provider | OpenClaw agent loop 文档列出 internal hooks 和 plugin hooks。锚点：`/code/openclaw/docs/concepts/agent-loop.md` | `ProviderProfile` registry 驱动 provider 能力。锚点：`/code/hermes-agent/providers/README.md` | 第三层才加入 hook/plugin/provider registry，避免第一版过度复杂。 |
| 沙箱 workspace | OpenClaw sandbox 可配置 mode/scope/backend，工具执行可进 sandbox。锚点：`/code/openclaw/docs/gateway/sandboxing.md` | README 声明多 terminal backends，包括 Docker/SSH/Modal/Daytona/Vercel Sandbox。锚点：`/code/hermes-agent/README.md` | 沙箱是高级能力，但接口要尽早抽象为 workspace boundary。 |

## 源码阅读路线

1. 第一层入口：读 `/code/openclaw/docs/concepts/architecture.md`，再读 `/code/hermes-agent/run_agent.py::AIAgent` 和 `run_conversation()`。
2. 第一层会话：读 `/code/openclaw/docs/concepts/agent.md` 的 `Sessions`，再在 `/code/hermes-agent/run_agent.py` 搜索 `_persist_session`、`session_id`、`conversation_history`。
3. 第一层 loop：读 `/code/openclaw/docs/concepts/agent-loop.md` 的 high-level steps，再读 `/code/hermes-agent/run_agent.py` 的 tool call validation、API retry、final response 分支。
4. 第二层 memory：读 `/code/openclaw/docs/concepts/memory.md`，再读 `/code/hermes-agent/agent/memory_manager.py`。
5. 第二层 queue/scheduler：读 `/code/openclaw/docs/concepts/queue.md`，再读 `/code/hermes-agent/cron/scheduler.py::_build_effective_prompt` 和 inactivity timeout 分支。
6. 第三层 extension：读 `/code/openclaw/docs/concepts/agent-loop.md` 的 hook points，再读 `/code/hermes-agent/providers/README.md`。
7. 第三层 sandbox：读 `/code/openclaw/docs/gateway/sandboxing.md`，再按需要追 Hermes terminal backend 实现。

## 最小复现抽象

以下三层是从前面章节抽象出的学习模型，不是 OpenClaw 或 Hermes 的精确实现。

第一层：最小可运行长期 agent。

```python
class Gateway:
    def accept(self, message) -> str: ...  # returns run_id

class CLI:
    def run_once(self, text: str) -> None: ...

class SessionStore:
    def resolve(self, source) -> str: ...
    def load(self, session_id: str) -> list[dict]: ...

class TranscriptStore:
    def append(self, session_id: str, event: dict) -> None: ...

class ToolRegistry:
    def schema(self) -> list[dict]: ...
    def call(self, name: str, args: dict) -> str: ...

class AgentLoop:
    def run(self, session_id: str, user_message: str) -> dict: ...
```

第一层只要求能从 Gateway/CLI 入口进入，解析 session，加载 transcript，组装 prompt，调用模型，执行工具，把 assistant/tool/lifecycle 事件写回 `TranscriptStore`。这个抽象对应 OpenClaw 的 Gateway + agent loop + session transcript，也对应 Hermes 的 `AIAgent.run_conversation()`。锚点：`/code/openclaw/docs/concepts/architecture.md`、`/code/openclaw/docs/concepts/agent-loop.md`、`/code/openclaw/docs/concepts/agent.md`、`/code/hermes-agent/run_agent.py`。

第二层：真正长生命周期能力。

```python
class MemoryStore:
    def write_fact(self, fact: str, evidence: str) -> None: ...

class MemoryRetriever:
    def search(self, query: str) -> list[str]: ...

class Scheduler:
    def due_jobs(self) -> list[dict]: ...

class DeliveryTarget:
    def send(self, payload: str) -> None: ...

class Lane:
    def enqueue(self, key: str, run) -> str: ...

class FailureRecovery:
    timeout_policy: object
    retry_policy: object
    def monitor(self, run_id: str) -> None: ...
```

第二层把“能聊天”升级为“能长期运行”：memory 让跨会话事实可召回；scheduler 让 agent 无人值守运行；delivery target 让结果回到用户所在平台；queue/lane 避免同一 session 并发写 transcript；failure recovery 处理 timeout、retry、interrupt、stuck diagnostics。锚点：`/code/openclaw/docs/concepts/memory.md`、`/code/openclaw/docs/concepts/queue.md`、`/code/openclaw/docs/concepts/retry.md`、`/code/hermes-agent/agent/memory_manager.py`、`/code/hermes-agent/cron/scheduler.py`、`/code/hermes-agent/run_agent.py`。

第三层：接近 OpenClaw/Hermes 的高级能力。

```python
class PluginManager:
    def run_hook(self, name: str, ctx: dict) -> dict: ...

class SkillCurator:
    def select(self, task: str) -> list[str]: ...
    def update_after_use(self, skill: str, feedback: str) -> None: ...

class MultiAgentRouter:
    def delegate(self, task: str, lane: str, workspace: str) -> str: ...

class SandboxedWorkspace:
    def open(self, session_id: str) -> str: ...

class ProviderRegistry:
    def resolve(self, requested: str, model: str) -> dict: ...
    def fallback(self, error: Exception) -> dict | None: ...
```

第三层引入 extension surface 和隔离：plugin/hook system 允许在 prompt、tool、message、session lifecycle 上插入策略；skill curator 让过程记忆可维护；multi-agent routing/delegation 让任务拆到不同 agent/workspace；sandboxed workspaces 降低工具执行 blast radius；provider registry/failover 把模型供应商差异从 agent loop 中隔离出来。锚点：`/code/openclaw/docs/concepts/agent-loop.md`、`/code/openclaw/docs/gateway/sandboxing.md`、`/code/hermes-agent/agent/prompt_builder.py`、`/code/hermes-agent/README.md`、`/code/hermes-agent/providers/README.md`、`/code/hermes-agent/run_agent.py`。

## 容易误解的点

- 本章不是实现计划；`Gateway`、`SessionStore`、`AgentLoop` 等类名是学习抽象，不是要求 OpenClaw/Hermes 必须存在同名类。锚点：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/design.md`。
- “最小可运行”不等于“长生命周期”；没有 memory、scheduler、queue/lane、failure recovery，只能算第一层。锚点：`/code/openclaw/docs/concepts/queue.md`、`/code/openclaw/docs/concepts/memory.md`、`/code/hermes-agent/cron/scheduler.py`。
- “有沙箱”不等于“绝对安全”；OpenClaw sandboxing 文档明确说它不是完美安全边界。锚点：`/code/openclaw/docs/gateway/sandboxing.md`。
- provider registry 不只是模型列表；Hermes 文档说明 auth、transport kwargs、model listing、runtime routing 都读取 `ProviderProfile`。锚点：`/code/hermes-agent/providers/README.md`。
- memory 不应被混同为 prompt 里的任意长文本；OpenClaw 把 durable memory、daily notes、dreaming 分开，Hermes 用 fenced memory context 标记召回记忆。锚点：`/code/openclaw/docs/concepts/memory.md`、`/code/hermes-agent/agent/memory_manager.py`。

## 待核实问题

- 本章三层抽象后续可以继续与 `00` 到 `10` 章的术语表合并，形成一张全书统一架构图。
- Hermes 的 `SessionStore`/`TranscriptStore` 等价实现需要进一步从 gateway/session、SQLite session store 和 persistence 路径核实。
- Hermes 的 terminal backend sandbox 边界尚未逐一阅读；本章只使用 README 的能力声明。
- OpenClaw 的 plugin/skill 更深源码路径仍可继续补充；本章第三层的 `PluginManager`、`SkillCurator` 已与 `06-skills.md` 和 `08-tools-plugins.md` 的第一版术语对齐，但还可以在后续深入章节中细化。
