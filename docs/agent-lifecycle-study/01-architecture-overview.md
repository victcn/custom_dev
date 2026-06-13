# 架构总览

> 参考版本：
> - OpenClaw：`/code/openclaw@ca31a705d02e42ffcfb2c5884bb55339a6d0cbdc`
> - Hermes Agent：`/code/hermes-agent@3d4297a59a8607ed24850524d229f5f42520d087`
> - Codex / Claude Code：作为入口、CLI/SDK/MCP/tool-use、memory、skills、automation、subagents 和 hooks/failure policy 的官方文档参照，核验日期同相关机制章节。

## 本章问题

学习抽象：本小册子把“长生命周期 agent”定义为一种能长期暴露入口、维持稳定 session、恢复或延续状态、跨会话使用记忆、通过 skills/process knowledge 复用经验、运行后台自动化、并通过 tools/plugins/provider 边界接入外部能力的系统。这个定义来自本小册子设计规格的机制清单和章节边界：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/design.md`。

这个定义不是说项目必须永远运行一个模型调用。更准确地说，长生命周期体现在外围系统：Gateway daemon、session store、workspace/context files、memory/skills、cron、provider registry、platform adapter、CLI/SDK host、MCP tool bridge、hooks 和 subagents。OpenClaw 的相关锚点见 `/code/openclaw/README.md`、`/code/openclaw/docs/concepts/architecture.md`、`/code/openclaw/docs/concepts/agent.md`；Hermes 的相关锚点见 `/code/hermes-agent/README.md`、`/code/hermes-agent/gateway/run.py`、`/code/hermes-agent/run_agent.py`、`/code/hermes-agent/providers/README.md`。Codex 和 Claude Code 在本章作为外部产品参照，详细协议和机制边界分散在 `02-gateway.md`、`03-agent-loop.md`、`05-memory.md`、`06-skills.md`、`07-automation.md`、`08-tools-plugins.md`、`09-multi-agent.md`、`10-failure-modes.md`。

## OpenClaw 怎么做

OpenClaw 的总体模型可以概括为：local-first Gateway + embedded agent runtime + workspace/session/memory/plugins。文档声明上，OpenClaw 是运行在用户自己设备上的 personal AI assistant，支持多个消息 channel，Gateway 是 control plane：`/code/openclaw/README.md`。

Gateway 层：architecture 文档说明单个 long-lived Gateway 拥有 WhatsApp、Telegram、Slack、Discord、Signal、iMessage、WebChat 等 messaging surfaces；control-plane clients 通过 WebSocket 连接，nodes 也通过同一 WebSocket server 连接并声明 `role: node`：`/code/openclaw/docs/concepts/architecture.md`。

Agent runtime 层：agent 文档说明 OpenClaw 运行 single embedded agent runtime，也就是 one agent process per Gateway；该 runtime 有自己的 workspace、bootstrap files 和 session store：`/code/openclaw/docs/concepts/agent.md`。同一文档还说明 `agents.defaults.workspace` 是工具和上下文的 `cwd`，并列出 `AGENTS.md`、`SOUL.md`、`TOOLS.md`、`BOOTSTRAP.md`、`IDENTITY.md`、`USER.md` 等注入 prompt 的 bootstrap files：`/code/openclaw/docs/concepts/agent.md`。

状态与记忆层：OpenClaw agent 文档说明 session transcripts 存在 `~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`，session ID 由 OpenClaw 选择且稳定；README 还把 skills、cron、sessions、tools、plugins 和 security model 作为 operator quick refs 或 docs by goal 的入口：`/code/openclaw/docs/concepts/agent.md`；`/code/openclaw/README.md`。

## Hermes 怎么做

Hermes 的总体模型可以概括为：Python `AIAgent` + messaging gateway/platform adapters + `MemoryManager` + skills/curator + cron/subagents + provider registry。文档声明上，Hermes 是 self-improving AI agent，强调从经验中创建 skills、改进 skills、持久化 knowledge、搜索过往 conversations、跨 session 建立 user model：`/code/hermes-agent/README.md`。

入口层：Hermes README 说明有两个入口：运行 `hermes` 启动 terminal UI，或运行 `hermes gateway` 从 Telegram、Discord、Slack、WhatsApp、Signal、Email 等消息平台对话：`/code/hermes-agent/README.md`。Gateway 代码中 `GatewayRunner` 的文档字符串说明它管理所有 platform adapters 的生命周期，并把消息路由到/来自 agent：`/code/hermes-agent/gateway/run.py`。

执行层：`run_agent.py` 中 `AIAgent` 是主 agent 类，`run_conversation` 是对话执行入口；`GatewayRunner._handle_message_with_agent` 会为来源创建/查找 session、构建 `SessionContext`、注入 context prompt，并在 agent 上调用 `run_conversation`：`/code/hermes-agent/run_agent.py`；`/code/hermes-agent/gateway/run.py`。

状态、记忆与 provider 层：`gateway/session.py` 定义 `SessionSource`、`SessionContext`、`SessionEntry` 和 `SessionStore`，负责来源、session key、session id、SQLite/JSONL fallback 等 metadata：`/code/hermes-agent/gateway/session.py`。`run_agent.py` 中 `AIAgent` 初始化路径集成 `MemoryManager`，并把 gateway 的 `user_id`、`gateway_session_key` 等传入 memory/provider 相关初始化：`/code/hermes-agent/run_agent.py`。Provider registry 的文档说明 `ProviderProfile` 是每个 inference provider 的单一声明，auth resolution、transport kwargs、model listing、runtime routing 都从 profile 读取：`/code/hermes-agent/providers/README.md`。

## Codex / Claude Code 参照

Codex 和 Claude Code 不作为本小册子的逐章源码对照对象，但它们能帮助校准“长生命周期 agent 不一定等于 Gateway daemon”。Codex CLI 的官方文档把 Codex 定位为本地 terminal 中运行的 coding agent；它可以通过 CLI 处理本地工作区，也提供 `codex mcp-server` 和 experimental `codex app-server` 这类扩展/本地协议入口。锚点：`https://developers.openai.com/codex/cli`；`https://developers.openai.com/codex/cli/reference`。

Claude Code 的官方 Agent SDK 文档说明 SDK 提供与 Claude Code 相同的 tools、agent loop 和 context management，让宿主程序用 Python/TypeScript 构建 coding agents；Anthropic 的 MCP connector 文档说明 Claude API 可以连接 remote MCP servers。锚点：`https://code.claude.com/docs/en/agent-sdk/overview`；`https://platform.claude.com/docs/en/agents-and-tools/mcp-connector`。

因此，读本章时可以把四类系统放在同一张心智图上：OpenClaw 是 local-first Gateway control plane；Hermes 是 Python `AIAgent` 加 gateway/adapters/API/webhook；Codex 是本地 CLI/app-server/MCP host；Claude Code 是 CLI/SDK/API/MCP host。它们共同说明：架构总览要先识别入口和扩展边界，再看内部 agent loop。

## 异同点表

| 维度 | OpenClaw | Hermes Agent | Codex / Claude Code 参照 |
|---|---|---|---|
| 定位 | 运行在用户设备上的 personal AI assistant；Gateway 是 control plane：`/code/openclaw/README.md` | self-improving AI agent；强调 learning loop、skills、memory、user model：`/code/hermes-agent/README.md` | coding-agent surface；更偏本地 CLI/SDK/API host，而不是统一 Gateway daemon。 |
| 运行入口 | `openclaw onboard --install-daemon` 安装 daemon，`openclaw gateway` 启动 Gateway，`openclaw agent` 与 assistant 对话：`/code/openclaw/README.md` | `hermes` 进入 CLI，`hermes gateway` 启动 messaging gateway：`/code/hermes-agent/README.md` | Codex CLI/app-server/MCP；Claude Code CLI/Agent SDK/MCP/API。 |
| Gateway/入口模型 | 单个 long-lived Gateway 拥有 messaging surfaces，clients/nodes 通过 WebSocket 接入：`/code/openclaw/docs/concepts/architecture.md` | `GatewayRunner` 管理 platform adapters 并路由消息到/来自 agent；另有 API/webhook adapter：`/code/hermes-agent/gateway/run.py` | 入口可以是 CLI prompt、SDK call、MCP request 或 local app-server，不一定叫 Gateway。 |
| 状态持久化 | session transcripts 为 `~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`：`/code/openclaw/docs/concepts/agent.md` | `SessionStore` 管理 session metadata，优先 SQLite，SQLite 不可用时 fallback 到 JSONL/session index：`/code/hermes-agent/gateway/session.py` | CLI/SDK host 通常由宿主负责 workspace、conversation/session 和持久化边界。 |
| 记忆 | OpenClaw agent 文档把 `AGENTS.md` 描述为 operating instructions + "memory"，并通过 bootstrap files 注入 Project Context：`/code/openclaw/docs/concepts/agent.md` | Hermes README 声明 agent-curated memory、FTS5 session search、Honcho user modeling；`AIAgent` 集成 `MemoryManager`：`/code/hermes-agent/README.md`；`/code/hermes-agent/run_agent.py` | Codex/Claude Code 侧更适合作为“repo guidance、context management、MCP data source”参照；本章不写成长期记忆实现事实。 |
| 技能/过程知识 | OpenClaw agent 文档列出 skills 搜索位置：workspace、project、personal、managed/local、bundled、extra dirs：`/code/openclaw/docs/concepts/agent.md` | Hermes README 声明 autonomous skill creation 和 skills self-improve；后续细节锚点为 `agent/curator.py`、`tools/skills_tool.py`：`/code/hermes-agent/README.md` | 可类比为 coding-agent 的 reusable instructions/workflows，但具体机制应回到各自官方文档核实。 |
| 自动化 | README 把 cron jobs、webhooks、Gmail Pub/Sub 放在 automation docs 入口，并把 cron 列为 first-class tools：`/code/openclaw/README.md` | README 声明 built-in cron scheduler with delivery to any platform：`/code/hermes-agent/README.md` | Codex/Claude Code 的自动化通常由宿主、CLI wrapper、CI、SDK 服务或 MCP 周边系统触发。 |
| 插件/provider | OpenClaw README 指向 plugins、model configuration、tools 和 channels；具体插件机制后续章节核实：`/code/openclaw/README.md` | Provider registry 明确由 `ProviderProfile` 统一 auth、transport、model listing、runtime routing：`/code/hermes-agent/providers/README.md` | MCP 是重要扩展协议；Claude API tool use 与 MCP 要区分。 |

## 源码阅读路线

OpenClaw 建议顺序：

1. `/code/openclaw/README.md`：先确定产品定位、Gateway、channels、tools、security defaults。
2. `/code/openclaw/docs/concepts/architecture.md`：看 long-lived Gateway、WebSocket clients、nodes、canvas host、pairing/auth。
3. `/code/openclaw/docs/concepts/agent.md`：看 embedded runtime、workspace、bootstrap files、sessions、skills。
4. 后续扩展到 `/code/openclaw/docs/concepts/agent-loop.md`、`/code/openclaw/docs/concepts/memory.md`、`/code/openclaw/docs/tools/skills.md`、`/code/openclaw/docs/automation/cron-jobs.md`。

Hermes 建议顺序：

1. `/code/hermes-agent/README.md` 与 `/code/hermes-agent/README.zh-CN.md`：先确定 self-improving、CLI/gateway、memory、skills、cron、subagents。
2. `/code/hermes-agent/gateway/run.py`：看 `GatewayRunner`、adapter 生命周期、message routing、agent cache。
3. `/code/hermes-agent/gateway/session.py`：看 `SessionSource`、`SessionContext`、`SessionStore`、`build_session_context_prompt`。
4. `/code/hermes-agent/run_agent.py`：看 `AIAgent`、`run_conversation`、`MemoryManager` 集成、provider routing。
5. `/code/hermes-agent/providers/README.md`：看 `ProviderProfile` 和 provider plugin contract。

Codex / Claude Code 参照顺序：

1. 先读 `02-gateway.md` 中的 Codex / Claude Code 小节，建立 CLI、SDK、MCP、app-server/API 的入口边界。
2. 如果关注工具扩展，跳到 `08-tools-plugins.md` 的 MCP 参照。
3. 如果关注最小复现，跳到 `11-minimal-reproduction.md` 中的 `IngressSurface`、`ToolBridge` 和 `AgentLoop` 抽象。

## 最小复现抽象

学习抽象：一个最小长生命周期 agent 架构至少需要八个接口：

1. `IngressSurface`：接收外部输入并转成统一事件，可能是 Gateway、HTTP API、Webhook、CLI、SDK call 或 MCP request。OpenClaw 对应 WebSocket Gateway：`/code/openclaw/docs/concepts/architecture.md`；Hermes 对应 `GatewayRunner` 和 `BasePlatformAdapter`：`/code/hermes-agent/gateway/run.py`；`/code/hermes-agent/gateway/platforms/base.py`；Codex/Claude Code 参照见 `02-gateway.md`。
2. `SessionStore`：从来源生成稳定 session key，并保存 transcript/metadata，OpenClaw session JSONL 见 `/code/openclaw/docs/concepts/agent.md`；Hermes `SessionStore` 见 `/code/hermes-agent/gateway/session.py`。
3. `ContextBuilder`：把 workspace files、session context、memory hints 注入 prompt，OpenClaw bootstrap files 见 `/code/openclaw/docs/concepts/agent.md`；Hermes `build_session_context_prompt` 见 `/code/hermes-agent/gateway/session.py`。
4. `AgentLoop`：调用模型、执行 tools、返回 final response，Hermes 锚点是 `AIAgent.run_conversation`：`/code/hermes-agent/run_agent.py`。
5. `MemoryLayer`：跨会话保留与召回信息，Hermes README 和 `MemoryManager` 集成提供起点：`/code/hermes-agent/README.md`；`/code/hermes-agent/run_agent.py`。
6. `SkillLayer`：加载过程记忆或操作手册，OpenClaw skill 路径见 `/code/openclaw/docs/concepts/agent.md`；Hermes skills 声明见 `/code/hermes-agent/README.md`。
7. `ToolBridge`：把本地工具、平台能力、MCP server 或 API tool use 接进 agent，OpenClaw node caps/commands 与 Hermes tool registry 是本地证据，Codex/Claude Code 的 MCP 参照见 `08-tools-plugins.md`。
8. `ProviderRegistry`：隔离模型 provider 差异，Hermes 明确用 `ProviderProfile`：`/code/hermes-agent/providers/README.md`；OpenClaw provider 细节留到 `08-tools-plugins.md` 核实。

## 容易误解的点

- “长生命周期”不等于“每个 session 都永久保留同一个 Python/Node 对象”。OpenClaw 文档强调 one agent process per Gateway 和 stable session transcript：`/code/openclaw/docs/concepts/agent.md`；Hermes `GatewayRunner` 虽然有 per-session `AIAgent` cache，但也有 cache cap/TTL 注释，不能理解为无限生命周期对象：`/code/hermes-agent/gateway/run.py`。
- OpenClaw 的 `Gateway` 是 WebSocket control plane；Hermes 的 gateway 是 messaging platform adapter runner。两者都叫 gateway，但入口形态和协议边界不同：`/code/openclaw/docs/concepts/architecture.md`；`/code/hermes-agent/gateway/run.py`。
- Codex/Claude Code 参照不是要把它们硬写成 Gateway 项目，而是提醒入口可能是 CLI、SDK、MCP 或 app-server/API。
- Hermes 的 provider plugin 不等于 OpenClaw 的 Gateway plugin。Hermes provider registry 的职责是 inference provider profile：`/code/hermes-agent/providers/README.md`；OpenClaw plugin/provider 细节需要后续章节从 OpenClaw plugin docs 和源码核实。

## 待核实问题

- OpenClaw 的 multi-agent routing 在 README 中被列为 highlight，但本章只读了 README 与 architecture/agent 文档，具体 route 到 isolated agents 的配置和源码实现待 `09-multi-agent.md` 核实：`/code/openclaw/README.md`。
- Hermes `MemoryManager`、skill curator、cron scheduler 的实现细节只在本章作为总览锚点提及，后续章节需阅读 `/code/hermes-agent/agent/memory_manager.py`、`/code/hermes-agent/agent/curator.py`、`/code/hermes-agent/cron/scheduler.py`。
