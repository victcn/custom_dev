# Gateway 入口、接口与控制面

> 参考版本：
> - OpenClaw：`/code/openclaw@ca31a705d02e42ffcfb2c5884bb55339a6d0cbdc`
> - Hermes Agent：`/code/hermes-agent@3d4297a59a8607ed24850524d229f5f42520d087`
> - Codex：OpenAI Codex manual 与官方 Codex CLI 文档，核验于 2026-06-13：`https://developers.openai.com/codex/codex-manual.md`；`https://developers.openai.com/codex/cli`
> - Claude Code：Anthropic 官方 Claude Code / Agent SDK / MCP 文档，核验于 2026-06-13：`https://code.claude.com/docs/en/agent-sdk/overview`；`https://platform.claude.com/docs/en/agents-and-tools/mcp-connector`

## 本章问题

本章关注“外部请求怎样进入 agent”：入口可能是长驻 Gateway、消息平台 adapter、HTTP API、Webhook、本地 CLI、SDK，或者 MCP client/server。不要先问“它是不是 WebSocket/gRPC”，而要先区分三层：

1. 用户或外部系统怎样触发一个 run。
2. 入口层怎样把外部请求归一化成 session、context、tool 权限和 delivery target。
3. agent loop 怎样被调用，以及输出怎样回到调用方。

OpenClaw 和 Hermes 都有 gateway 概念，但边界不同。OpenClaw 是统一 WebSocket control plane；Hermes 是多平台 adapter runner，并额外提供 OpenAI-compatible HTTP API 与 webhook adapter。Codex 和 Claude Code 更接近本地 coding agent/SDK surface：它们的主要入口是 CLI、IDE/应用或 SDK，扩展协议主要是 MCP，而不是一个类似 OpenClaw 的统一 Gateway daemon。

## OpenClaw 怎么做

OpenClaw 的 Gateway 是 daemon/control plane。architecture 文档说明：single long-lived `Gateway` 拥有 WhatsApp、Telegram、Slack、Discord、Signal、iMessage、WebChat 等 messaging surfaces；control-plane clients 通过 WebSocket 连接到默认 `127.0.0.1:18789`；nodes 也通过同一 WebSocket server 连接，但在 `connect` 中声明 `role: node` 和 caps/commands：`/code/openclaw/docs/concepts/architecture.md`。

协议形态是 WebSocket RPC。Gateway protocol 文档说明 transport 是 WebSocket text frames with JSON payloads；第一帧必须是 `connect` request；握手后 request/response/event 分别是 `{type:"req", id, method, params}`、`{type:"res", id, ok, payload|error}`、`{type:"event", event, payload, seq?, stateVersion?}`：`/code/openclaw/docs/gateway/protocol.md`。

clients/nodes 的边界在握手中确定。Protocol 文档把 `operator` 定义为 control plane client，把 `node` 定义为 capability host，并要求 node 在 `connect` 中声明 `caps`、`commands`、`permissions`；Gateway treats these as claims and enforces server-side allowlists：`/code/openclaw/docs/gateway/protocol.md`。

pairing/auth 是连接安全边界的一部分。architecture 文档说明所有 WS clients 包含 device identity，新 device IDs 需要 pairing approval，Gateway 为后续连接签发 device token，非本地连接仍需 explicit approval，Gateway auth 仍适用于所有连接：`/code/openclaw/docs/concepts/architecture.md`。Protocol 文档还说明 shared-secret auth 使用 `connect.params.auth.token` 或 `connect.params.auth.password`，Tailscale/trusted-proxy 等 identity-bearing modes 可从 request headers 满足 auth：`/code/openclaw/docs/gateway/protocol.md`。

Gateway 与 agent loop 的关系是 RPC 入口触发 agent run 并推送事件。architecture 文档的 sequence diagram 展示 `req:agent` 返回 accepted ack，随后 Gateway 推送 streaming `event:agent`，最后返回 final `res:agent`：`/code/openclaw/docs/concepts/architecture.md`。这说明 Gateway 不只是消息转发器，还承担 agent run 的控制面和事件流出口；agent loop 细节留给后续 `03-agent-loop.md`，入口锚点是 `/code/openclaw/docs/concepts/agent-loop.md`。

## Hermes 怎么做

Hermes 的 gateway 是 platform adapter runner，而不是 OpenClaw 那种统一公开给 clients/nodes 的 WebSocket RPC 控制面。README 说明用户可以运行 `hermes gateway` 并从 Telegram、Discord、Slack、WhatsApp、Signal 或 Email 与 agent 对话：`/code/hermes-agent/README.md`。代码中 `GatewayRunner` 的文档字符串说明它管理所有 platform adapters 生命周期，并把 messages 路由到/来自 agent：`/code/hermes-agent/gateway/run.py`。

platform adapter 是协议适配边界。`BasePlatformAdapter` 的文档字符串说明子类负责 platform-specific logic：connecting/authenticating、receiving messages、sending responses、handling media；`MessageEvent` 是所有 adapter 产出的 normalized representation，并包含 `text`、`message_type`、`source`、`raw_message`、`message_id`、`media_urls`、`reply_to_message_id`、`auto_skill`、`channel_prompt` 等字段：`/code/hermes-agent/gateway/platforms/base.py`。

Hermes 也有可直接对外暴露的 HTTP 接口，但这是 `api_server` adapter，不是整个 gateway 的统一协议。`gateway/platforms/api_server.py` 文件注释列出 OpenAI-compatible endpoints：`POST /v1/chat/completions`、`POST /v1/responses`、`GET /v1/responses/{response_id}`、`DELETE /v1/responses/{response_id}`、`GET /v1/models`、`GET /v1/capabilities`、`POST /v1/runs`、`GET /v1/runs/{run_id}`、`GET /v1/runs/{run_id}/events`、`POST /v1/runs/{run_id}/approval`、`POST /v1/runs/{run_id}/stop`、`GET /health`、`GET /health/detailed`；默认 host/port 是 `127.0.0.1:8642`：`/code/hermes-agent/gateway/platforms/api_server.py`。

Hermes 的流式接口主要走 HTTP SSE。`api_server` 的 capabilities endpoint 声明 `chat_completions_streaming`、`responses_streaming`、`run_events_sse`、`tool_progress_events`、`approval_events` 等能力，并注册 `GET /v1/runs/{run_id}/events` 作为 run events stream：`/code/hermes-agent/gateway/platforms/api_server.py`。

Webhook 是另一类 HTTP 入口。`gateway/platforms/webhook.py` 声明它运行 aiohttp HTTP server，接收外部服务 POST，验证 HMAC signature，把 payload 转成 agent prompt，并把响应投递到 GitHub comment、Telegram、Discord、Slack 等目标。它默认 `0.0.0.0:8644`，路由是 `GET /health` 和 `POST /webhooks/{route_name}`：`/code/hermes-agent/gateway/platforms/webhook.py`。

session/source 是路由上下文的核心。`SessionSource` 描述消息来源，字段包含 `platform`、`chat_id`、`chat_type`、`user_id`、`thread_id`、`guild_id`、`parent_chat_id`、`message_id` 等；它的 docstring 明确说这些信息用于 route responses、inject context into system prompt、track origin for cron job delivery：`/code/hermes-agent/gateway/session.py`。`SessionContext` 包含 `source`、`connected_platforms`、`home_channels`、`shared_multi_user_session`、`session_key`、`session_id` 等，并被 `build_session_context_prompt` 转为动态 system prompt：`/code/hermes-agent/gateway/session.py`。

Gateway 与 agent loop 的关系在 `GatewayRunner._handle_message_with_agent` 和内部 run path 中体现：它先 `session_store.get_or_create_session(source)`，再 `build_session_context(source, config, session_entry)`，然后 `build_session_context_prompt(context, redact_pii=...)`，最后创建或复用 `AIAgent` 并调用 `agent.run_conversation(..., conversation_history=agent_history, task_id=session_id)`：`/code/hermes-agent/gateway/run.py`；`/code/hermes-agent/gateway/session.py`；`/code/hermes-agent/run_agent.py`。

## Codex 怎么做

Codex CLI 的主要入口是本地终端，而不是一个默认公开的 Gateway。OpenAI 官方文档把 Codex CLI 定义为可以在本地 terminal 中运行的 coding agent，可以读取、修改并运行 selected directory 中的代码：`https://developers.openai.com/codex/cli`。这意味着 Codex 的用户侧入口通常是 CLI/IDE/app/cloud task surface，而不是让第三方 client 通过固定端口连接一个 Codex daemon。

Codex 的交互协议可以分两类看。第一类是用户侧 CLI：`codex` 启动交互 TUI，`codex "prompt"` 可从命令行给初始任务；CLI 在本机工作区内读取文件、编辑文件、运行命令，并根据 sandbox、approval、config 和 repo guidance 执行任务。第二类是扩展侧 MCP：Codex CLI reference 明确有 `codex mcp-server`，可让 Codex 作为 MCP server over stdio，被其他工具连接：`https://developers.openai.com/codex/cli/reference`。

Codex 也可以作为 MCP client 连接外部 MCP servers。Codex MCP 文档把 MCP server 配置放在 `config.toml` 的 `mcp_servers` 表下，并支持 `stdio` 与 `streamable_http` 两类传输；这说明 Codex 的外部工具/数据扩展面是 MCP，而不是每个工具都写成 Codex 私有插件协议：`https://developers.openai.com/codex/mcp`。

Codex 还有 experimental `codex app-server`。官方 reference 说明它用于 local development/debugging，可通过 `stdio://`、`ws://IP:PORT`、`unix://`、`unix://PATH` 或 `off` 配置本地 transport；WebSocket 客户端可配置 capability token 或 signed bearer token auth：`https://developers.openai.com/codex/cli/reference`。这和 OpenClaw Gateway 有相似的“本地 app-server transport”形态，但定位更窄：主要服务 Codex app/server 调试、remote-control 和本地协议 client，而不是 OpenClaw 那种统一消息平台、operator、node、agent run 的产品级控制面。

因此在协议对比里，Codex 不应被简单归类成 OpenClaw 式 WebSocket Gateway。它更像“本地 agent host + optional app-server transport + MCP client/server + OpenAI 模型 API backend”。如果要让别的系统驱动 Codex，优先看 `codex mcp-server`、`codex app-server` 或上层自动化/SDK；如果要让 Codex 调外部系统，优先看 MCP server 配置。

## Claude Code 怎么做

Claude Code 的主要入口也是 coding-agent surface，而不是一个默认公开的 WebSocket/gRPC Gateway。Anthropic 的 Claude Agent SDK 文档说明：Agent SDK 提供与 Claude Code 相同的 tools、agent loop 和 context management，并可用 Python/TypeScript 编程方式构建能读文件、运行命令、搜索网页、编辑代码的 agents：`https://code.claude.com/docs/en/agent-sdk/overview`。

Claude Code 的扩展协议核心同样是 MCP。Anthropic 的 MCP 文档把 Model Context Protocol 描述为连接 agent 与外部系统的开放标准；Claude API 的 MCP connector 文档说明 Messages API 可以直接连接 remote MCP servers，不需要单独 MCP client：`https://www.anthropic.com/news/model-context-protocol`；`https://platform.claude.com/docs/en/agents-and-tools/mcp-connector`。

Claude Code/Agent SDK 还要和 Claude API 的 tool use 区分。Claude API tool use 是 HTTP Messages API 中的 structured tool-call contract：Claude 决定何时调用工具，应用或 Anthropic server 执行对应工具。MCP 则把工具定义、发现和调用放进一个 client/server 协议里，适合连接长期存在的外部系统或工具服务：`https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview`。

因此在协议对比里，Claude Code 也不应被写成 OpenClaw 式 Gateway。它的主要外部接口是 CLI/SDK/API；工具扩展面是 MCP 和 Claude API tool use；长驻服务、HTTP endpoint 或 WebSocket endpoint 是否存在，取决于你用 Agent SDK 自己封装出的宿主程序，而不是 Claude Code 默认提供的统一对外网关。

## 异同点表

| 维度 | OpenClaw | Hermes | Codex | Claude Code | 学习结论 |
|---|---|---|---|---|---|
| 入口形态 | 单个 long-lived Gateway daemon，control-plane clients 和 nodes 通过同一 WebSocket server 接入：`/code/openclaw/docs/concepts/architecture.md` | `GatewayRunner` 启动 messaging platform adapters；另有 `api_server` 和 `webhook` adapter：`/code/hermes-agent/gateway/run.py`；`/code/hermes-agent/gateway/platforms/api_server.py` | 本地 CLI/IDE/app/cloud task surface；CLI 可作为 MCP server；experimental app-server 可供本地协议 client 调试/开发：`https://developers.openai.com/codex/cli`；`https://developers.openai.com/codex/cli/reference` | CLI/Agent SDK/API surface；SDK 可编程复用 Claude Code agent loop：`https://code.claude.com/docs/en/agent-sdk/overview` | “入口层”不一定叫 Gateway；先区分 daemon、adapter、HTTP API、CLI、SDK、MCP、local app-server。 |
| 对外协议 | WebSocket text JSON frames；`connect` 后使用 typed req/res/event RPC：`/code/openclaw/docs/gateway/protocol.md` | HTTP/OpenAI-compatible API、HTTP webhook、各平台自有协议；run events 用 SSE：`/code/hermes-agent/gateway/platforms/api_server.py`；`/code/hermes-agent/gateway/platforms/webhook.py` | CLI 本地交互；MCP server over stdio；MCP client 支持 stdio/streamable HTTP；experimental app-server 支持 stdio、WebSocket、Unix socket：`https://developers.openai.com/codex/cli/reference`；`https://developers.openai.com/codex/mcp` | Claude API 是 HTTP Messages API；MCP connector 连接 remote MCP servers；Agent SDK 由宿主程序决定对外协议：`https://platform.claude.com/docs/en/agents-and-tools/mcp-connector` | 不要把所有 agent 入口都归结为 WS/HTTP/gRPC 三选一；MCP 已成为 coding-agent 工具扩展的重要协议。 |
| clients/nodes/工具扩展 | `operator` 是 control-plane client，`node` 是 capability host，node 声明 `caps`/`commands`/`permissions`：`/code/openclaw/docs/gateway/protocol.md` | adapter 子类负责连接、收消息、发消息、媒体处理；HTTP API 用 Bearer token：`/code/hermes-agent/gateway/platforms/base.py`；`/code/hermes-agent/gateway/platforms/api_server.py` | MCP server/client；repo guidance、skills、plugins、hooks、sandbox 是本地 host 的控制面概念 | MCP servers、Claude API tool use、SDK options 是扩展边界 | 设备能力 host、消息平台 adapter、MCP server、API tool 都是不同扩展面。 |
| 会话上下文 | agent runtime 文档说明 session ID 由 OpenClaw 选择且 transcript 存在 JSONL；Gateway protocol 本章只覆盖入口：`/code/openclaw/docs/concepts/agent.md` | `SessionSource`、`SessionContext`、`build_session_key`、`SessionStore.get_or_create_session` 显式把来源转为 session/context：`/code/hermes-agent/gateway/session.py` | CLI session、工作区、repo guidance 和 conversation context 由 Codex host 管理；不是 WS frame 中的 session key | SDK/API 调用方可自己管理 session/context；Claude Code host 管理本地工作上下文 | session 是入口归一化的产物，不一定表现为网络协议字段。 |
| 安全边界 | device identity、pairing approval、device token、shared-secret auth、role/scopes、server-side allowlists：`/code/openclaw/docs/concepts/architecture.md`；`/code/openclaw/docs/gateway/protocol.md` | allowlists/allow-all、DM pairing、dangerous command approval；API server 用 Bearer token，webhook 用 HMAC：`/code/hermes-agent/gateway/run.py`；`/code/hermes-agent/gateway/platforms/api_server.py`；`/code/hermes-agent/gateway/platforms/webhook.py` | sandbox、approval、config、MCP server trust、local filesystem permissions | SDK/API keys、MCP server trust、tool execution policy、host process sandbox | 长期入口必须在 agent loop 之前做身份、授权、工具权限和高风险动作审批。 |
| 与 agent loop 的关系 | `req:agent` 触发 accepted ack、streaming `event:agent`、final `res:agent`：`/code/openclaw/docs/concepts/architecture.md` | `_handle_message_with_agent` 构造 session context 和 history，创建/复用 `AIAgent`，调用 `run_conversation`：`/code/hermes-agent/gateway/run.py`；`/code/hermes-agent/run_agent.py` | CLI/host 驱动 agent loop，MCP server 让其他工具把 Codex 当能力调用 | SDK 直接暴露 agent loop/context management 给宿主程序 | Gateway/API/CLI/SDK 都只是 agent loop 的入口；不要把入口层和模型/tool loop 混成一个概念。 |
| delivery context | Gateway events/server push 是 WebSocket 事件流的一部分：`/code/openclaw/docs/concepts/architecture.md` | `SessionSource` 记录 origin，`DeliveryRouter` 通过 adapter `send` 投递到 target：`/code/hermes-agent/gateway/session.py`；`/code/hermes-agent/gateway/delivery.py` | 输出通常回到 CLI/TUI、编辑 diff、命令结果或 MCP client | 输出回到 CLI/SDK/API caller，由宿主决定是否转发到 UI、HTTP、队列或消息平台 | 投递目标应保留为结构化上下文，避免 agent 只凭自然语言猜回复位置。 |

## 源码与文档阅读路线

OpenClaw：

1. `/code/openclaw/docs/concepts/architecture.md`：读概览、组件与流程、连接生命周期、wire protocol、pairing 与 local trust 相关段落。
2. `/code/openclaw/docs/gateway/protocol.md`：读 Transport、Handshake、Framing、Roles + scopes、Caps/commands/permissions。
3. `/code/openclaw/docs/concepts/agent.md`：补上 Gateway 后面的 embedded runtime、workspace、sessions。
4. 后续章节再读 `/code/openclaw/docs/concepts/agent-loop.md` 和 queue/session 相关文档。

Hermes：

1. `/code/hermes-agent/gateway/run.py`：先读 `GatewayRunner.__init__`、`GatewayRunner.start`、`GatewayRunner._handle_message`、`GatewayRunner._handle_message_with_agent`。
2. `/code/hermes-agent/gateway/platforms/base.py`：读 `MessageEvent`、`BasePlatformAdapter`、`BasePlatformAdapter.handle_message`。
3. `/code/hermes-agent/gateway/platforms/api_server.py`：读 endpoint 列表、auth、SSE、`/v1/runs` 和 `/v1/chat/completions`。
4. `/code/hermes-agent/gateway/platforms/webhook.py`：读 HMAC、rate limit、idempotency、`/webhooks/{route_name}`。
5. `/code/hermes-agent/gateway/session.py`：读 `SessionSource`、`SessionContext`、`build_session_context_prompt`、`build_session_key`、`SessionStore.get_or_create_session`、`build_session_context`。
6. `/code/hermes-agent/gateway/delivery.py`：读 `DeliveryRouter`，理解 background/cron/tool delivery 如何回到 adapter。
7. `/code/hermes-agent/run_agent.py`：定位 `AIAgent` 和 `run_conversation`，把 gateway 输入接到 agent loop。

Codex：

1. `https://developers.openai.com/codex/cli`：先确认 Codex CLI 的定位和本地工作区能力。
2. `https://developers.openai.com/codex/cli/features`：读 interactive mode、CLI prompt、editing、command execution 等用户入口。
3. `https://developers.openai.com/codex/cli/reference`：读 `codex mcp-server`、`codex app-server`、`--remote`、`--listen`、WebSocket auth 相关选项。
4. `https://developers.openai.com/codex/mcp`：读 `mcp_servers`、stdio/streamable HTTP、MCP tool approval 和 timeout 相关内容。
5. `https://developers.openai.com/codex/config-reference`：读 sandbox、approval、config 相关内容。

Claude Code：

1. `https://code.claude.com/docs/en/agent-sdk/overview`：读 Agent SDK 如何复用 Claude Code 的 tools、agent loop、context management。
2. `https://platform.claude.com/docs/en/agents-and-tools/mcp-connector`：读 Messages API 如何连接 remote MCP servers。
3. `https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview`：区分 Claude API tool use 与 MCP。
4. `https://www.anthropic.com/news/model-context-protocol`：读 MCP 的协议定位和生态背景。

## 最小复现抽象

学习抽象：入口层最小复现可以分成八个对象：

1. `IngressSurface`：接收外部输入，可以是 WebSocket server、HTTP API、webhook、CLI prompt、SDK call 或 MCP request。
2. `TransportAdapter`：把 WS frame、HTTP JSON、SSE、webhook payload、platform update、CLI input 或 MCP request 归一化为内部事件。
3. `SessionRouter`：从 source 生成 session key/session id，并决定这个消息进入哪个 agent/session。Hermes 的现成锚点是 `build_session_key` 和 `SessionStore.get_or_create_session`：`/code/hermes-agent/gateway/session.py`；OpenClaw session store 锚点是 `/code/openclaw/docs/concepts/agent.md`。
4. `DeliveryTarget`：结构化记录回复或后台任务结果应该投递到哪里。Hermes 的锚点是 `SessionSource`、`build_session_context_prompt` 和 `DeliveryRouter`：`/code/hermes-agent/gateway/session.py`；`/code/hermes-agent/gateway/delivery.py`。
5. `AuthPairingStore`：在进入 agent 前验证 token/device/user allowlist，并保存 pairing approval 或 device token。OpenClaw 的锚点是 pairing/auth 文档：`/code/openclaw/docs/concepts/architecture.md`；Hermes 的锚点是 `_is_user_authorized`、API Bearer token 和 webhook HMAC。
6. `ToolBridge`：把外部工具系统接进 agent。OpenClaw 是 node caps/commands；Hermes 是 platform/tool adapters；Codex 和 Claude Code 的关键协议是 MCP，也可以是 API tool use。
7. `AgentInvoker`：把 context prompt、history、message 交给 agent loop。Hermes 的锚点是 `_handle_message_with_agent` 调用 `run_conversation`：`/code/hermes-agent/gateway/run.py`；`/code/hermes-agent/run_agent.py`。Claude Agent SDK 则把这个边界暴露给 Python/TypeScript 宿主程序。
8. `StreamPublisher`：把中间事件和最终结果发回调用方。OpenClaw 用 WS events；Hermes API server 用 SSE；CLI/SDK 用本地 stdout、callback、iterator 或宿主程序自定义通道。

## 容易误解的点

- OpenClaw 的 Gateway protocol 是统一 WS RPC；Hermes 的 gateway 不是同一种协议层，而是多平台 adapter runner，另外可选 HTTP API 和 webhook adapter。这个差异由 OpenClaw protocol 文档和 Hermes `BasePlatformAdapter`/`GatewayRunner`/`api_server` 文档字符串共同支撑：`/code/openclaw/docs/gateway/protocol.md`；`/code/hermes-agent/gateway/platforms/base.py`；`/code/hermes-agent/gateway/run.py`；`/code/hermes-agent/gateway/platforms/api_server.py`。
- Hermes 有 HTTP 接口，但不能因此说 Hermes gateway 的协议就是 HTTP。Telegram、Discord、Slack、Email、Webhook、API Server 都是不同 adapter surface。
- Codex 和 Claude Code 的主要用户入口不是公开 Gateway。Codex 虽有 experimental `app-server` WebSocket/stdio/Unix socket transport，但定位是本地 app-server development/debugging 和 remote-control，并不等同于 OpenClaw 的统一消息 Gateway。做系统集成时优先看 MCP/SDK/API 或官方 app-server 文档，而不是假设有一个稳定公开的默认 WebSocket 端口。
- MCP 不是模型推理 API。MCP 连接 agent host 和外部工具/数据源；OpenAI/Anthropic 的模型 API 仍是模型调用层。
- `SessionSource` 不是 transcript 本身。它描述消息来源和投递上下文；session metadata/transcript 由 `SessionStore` 和 `SessionEntry` 管理：`/code/hermes-agent/gateway/session.py`。
- OpenClaw 的 `node` 不是普通聊天用户。Protocol 文档把 `node` 定义为 capability host，需要声明 caps/commands/permissions，且 Gateway server-side enforce allowlists：`/code/openclaw/docs/gateway/protocol.md`。

## 待核实问题

- OpenClaw Gateway 的具体 TypeBox schema、JSON Schema codegen 和 server implementation 尚未展开阅读；本章只依据 docs 描述协议与架构：`/code/openclaw/docs/gateway/protocol.md`。
- Hermes 的各平台 adapter 子类如 Telegram、Discord、Slack、WhatsApp 的实际 auth/polling/webhook 差异尚未逐一核实；本章只核实 `BasePlatformAdapter` 抽象、`GatewayRunner` 生命周期、`api_server` 和 `webhook` adapter：`/code/hermes-agent/gateway/platforms/base.py`；`/code/hermes-agent/gateway/run.py`；`/code/hermes-agent/gateway/platforms/api_server.py`；`/code/hermes-agent/gateway/platforms/webhook.py`。
- Codex manual 已在 2026-06-13 成功核验：`https://developers.openai.com/codex/codex-manual.md`；本章仍保留 Codex CLI 页面作为更易读的入口文档。
- Claude Code 本章依据公开官方文档描述 CLI/SDK/API/MCP 入口；未读取本地 Claude Code 源码，因此不写具体内部类名或私有协议。
