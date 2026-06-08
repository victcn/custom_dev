# Gateway 入口与控制面

> 参考版本：
> - OpenClaw：`/code/openclaw@ca31a705d02e42ffcfb2c5884bb55339a6d0cbdc`
> - Hermes Agent：`/code/hermes-agent@3d4297a59a8607ed24850524d229f5f42520d087`

## 本章问题

Gateway 解决的是长生命周期 agent 的“入口持久化”和“边界转换”问题：外部平台、设备、CLI 或后台任务不会直接等同于 agent loop，它们先进入一个长驻入口，由入口完成认证、配对、路由、会话识别、上下文构造、队列/中断处理和最终投递。OpenClaw 的 Gateway 入口由 WebSocket protocol 和 daemon 架构描述：`/code/openclaw/docs/concepts/architecture.md`；`/code/openclaw/docs/gateway/protocol.md`。Hermes 的 gateway 入口由 `GatewayRunner`、`BasePlatformAdapter`、`SessionSource` 和 `SessionContext` 组合实现：`/code/hermes-agent/gateway/run.py`；`/code/hermes-agent/gateway/platforms/base.py`；`/code/hermes-agent/gateway/session.py`。

## OpenClaw 怎么做

OpenClaw 的 Gateway 是 daemon/control plane。architecture 文档说明：single long-lived `Gateway` 拥有 WhatsApp、Telegram、Slack、Discord、Signal、iMessage、WebChat 等 messaging surfaces；control-plane clients 通过 WebSocket 连接到默认 `127.0.0.1:18789`；nodes 也通过同一 WebSocket server 连接，但在 `connect` 中声明 `role: node` 和 caps/commands：`/code/openclaw/docs/concepts/architecture.md`。

协议形态是 WebSocket RPC。Gateway protocol 文档说明 transport 是 WebSocket text frames with JSON payloads；第一帧必须是 `connect` request；握手后 request/response/event 分别是 `{type:"req", id, method, params}`、`{type:"res", id, ok, payload|error}`、`{type:"event", event, payload, seq?, stateVersion?}`：`/code/openclaw/docs/gateway/protocol.md`。

clients/nodes 的边界在握手中确定。Protocol 文档把 `operator` 定义为 control plane client，把 `node` 定义为 capability host，并要求 node 在 `connect` 中声明 `caps`、`commands`、`permissions`；Gateway treats these as claims and enforces server-side allowlists：`/code/openclaw/docs/gateway/protocol.md`。

pairing/auth 是连接安全边界的一部分。architecture 文档说明所有 WS clients 包含 device identity，新 device IDs 需要 pairing approval，Gateway 为后续连接签发 device token，非本地连接仍需 explicit approval，Gateway auth 仍适用于所有连接：`/code/openclaw/docs/concepts/architecture.md`。Protocol 文档还说明 shared-secret auth 使用 `connect.params.auth.token` 或 `connect.params.auth.password`，Tailscale/trusted-proxy 等 identity-bearing modes 可从 request headers 满足 auth：`/code/openclaw/docs/gateway/protocol.md`。

Gateway 与 agent loop 的关系是 RPC 入口触发 agent run 并推送事件。architecture 文档的 sequence diagram 展示 `req:agent` 返回 accepted ack，随后 Gateway 推送 streaming `event:agent`，最后返回 final `res:agent`：`/code/openclaw/docs/concepts/architecture.md`。这说明 Gateway 不只是消息转发器，还承担 agent run 的控制面和事件流出口；agent loop 细节留给后续 `03-agent-loop.md`，入口锚点是 `/code/openclaw/docs/concepts/agent-loop.md`。

## Hermes 怎么做

Hermes 的 gateway 是 platform adapter runner，而不是 OpenClaw 那种统一公开给 clients/nodes 的 WebSocket RPC 控制面。README 说明用户可以运行 `hermes gateway` 并从 Telegram、Discord、Slack、WhatsApp、Signal 或 Email 与 agent 对话：`/code/hermes-agent/README.md`。代码中 `GatewayRunner` 的文档字符串说明它管理所有 platform adapters 生命周期，并把 messages 路由到/来自 agent：`/code/hermes-agent/gateway/run.py`。

platform adapter 是协议适配边界。`BasePlatformAdapter` 的文档字符串说明子类负责 platform-specific logic：connecting/authenticating、receiving messages、sending responses、handling media；`MessageEvent` 是所有 adapter 产出的 normalized representation，并包含 `text`、`message_type`、`source`、`raw_message`、`message_id`、`media_urls`、`reply_to_message_id`、`auto_skill`、`channel_prompt` 等字段：`/code/hermes-agent/gateway/platforms/base.py`。

session/source 是路由上下文的核心。`SessionSource` 描述消息来源，字段包含 `platform`、`chat_id`、`chat_type`、`user_id`、`thread_id`、`guild_id`、`parent_chat_id`、`message_id` 等；它的 docstring 明确说这些信息用于 route responses、inject context into system prompt、track origin for cron job delivery：`/code/hermes-agent/gateway/session.py`。`SessionContext` 包含 `source`、`connected_platforms`、`home_channels`、`shared_multi_user_session`、`session_key`、`session_id` 等，并被 `build_session_context_prompt` 转为动态 system prompt：`/code/hermes-agent/gateway/session.py`。

session key 和持久化由 `SessionStore` 管理。`build_session_key` 根据 `SessionSource` 生成 deterministic session key，并区分 DM、group/channel、thread、per-user isolation 等规则；`SessionStore.get_or_create_session` 创建或复用 `SessionEntry`，生成 `session_id`，保存 session index，并在 SQLite 可用时创建 session 记录，SQLite 不可用时有 fallback 路径：`/code/hermes-agent/gateway/session.py`。

安全边界主要体现在 user allowlists、pairing 和 command approval。`GatewayRunner.start` 在启动时检查 allowlist/allow-all 环境变量并发出无 allowlist 警告；`GatewayRunner._handle_message` 会在进入 agent 前调用 `_is_user_authorized`，未授权 DM 可进入 pairing code 流程；危险命令审批通过 `tools.approval.register_gateway_notify` 从 agent thread 桥接到 adapter 发送 `/approve`/`/deny` 提示：`/code/hermes-agent/gateway/run.py`。

Gateway 与 agent loop 的关系在 `GatewayRunner._handle_message_with_agent` 和内部 run path 中体现：它先 `session_store.get_or_create_session(source)`，再 `build_session_context(source, config, session_entry)`，然后 `build_session_context_prompt(context, redact_pii=...)`，最后创建或复用 `AIAgent` 并调用 `agent.run_conversation(..., conversation_history=agent_history, task_id=session_id)`：`/code/hermes-agent/gateway/run.py`；`/code/hermes-agent/gateway/session.py`；`/code/hermes-agent/run_agent.py`。

## 异同点表

| 维度 | OpenClaw | Hermes | 学习结论 |
|---|---|---|---|
| 入口形态 | 单个 long-lived Gateway daemon，control-plane clients 和 nodes 通过同一 WebSocket server 接入：`/code/openclaw/docs/concepts/architecture.md` | `GatewayRunner` 启动各 messaging platform adapters，用户从 Telegram/Discord/Slack/WhatsApp/Signal/Email 等平台进入：`/code/hermes-agent/README.md`；`/code/hermes-agent/gateway/run.py` | 最小模型需要一个长期入口，但入口可以是统一控制面，也可以是平台适配器 runner。 |
| 协议/适配 | WebSocket text JSON frames；`connect` 后使用 typed req/res/event RPC：`/code/openclaw/docs/gateway/protocol.md` | 各平台 adapter 把平台 update 归一化为 `MessageEvent`，再交给 `GatewayRunner._handle_message`：`/code/hermes-agent/gateway/platforms/base.py`；`/code/hermes-agent/gateway/run.py` | 协议层要和 agent loop 解耦，先归一化为内部事件或请求。 |
| clients/nodes | `operator` 是 control-plane client，`node` 是 capability host，node 声明 `caps`/`commands`/`permissions`：`/code/openclaw/docs/gateway/protocol.md` | 没有同等的统一 WS `node` 角色；adapter 子类负责连接、收消息、发消息、媒体处理：`/code/hermes-agent/gateway/platforms/base.py` | 设备能力 host 和消息平台 adapter 是两种不同扩展面，不能混同。 |
| 会话上下文 | agent runtime 文档说明 session ID 由 OpenClaw 选择且 transcript 存在 JSONL；Gateway protocol 本章只覆盖入口，session 细节留后续章节：`/code/openclaw/docs/concepts/agent.md` | `SessionSource`、`SessionContext`、`build_session_key`、`SessionStore.get_or_create_session` 显式把来源转为 session/context：`/code/hermes-agent/gateway/session.py` | Gateway 层至少要把外部来源解析成稳定 session 身份和可注入上下文。 |
| 安全边界 | device identity、pairing approval、device token、shared-secret auth、role/scopes、server-side allowlists：`/code/openclaw/docs/concepts/architecture.md`；`/code/openclaw/docs/gateway/protocol.md` | allowlists/allow-all 检查、unauthorized DM pairing flow、dangerous command approval bridge：`/code/hermes-agent/gateway/run.py` | 长期入口必须在 agent loop 之前做身份、授权和高风险动作审批。 |
| 与 agent loop 的关系 | `req:agent` 触发 accepted ack、streaming `event:agent`、final `res:agent`：`/code/openclaw/docs/concepts/architecture.md` | `_handle_message_with_agent` 构造 session context 和 history，创建/复用 `AIAgent`，调用 `run_conversation`：`/code/hermes-agent/gateway/run.py`；`/code/hermes-agent/run_agent.py` | Gateway 负责接入和调度 run，不应该把模型/tool loop 细节塞进平台 adapter。 |
| delivery context | Gateway events/server push 是 WebSocket 事件流的一部分；channel delivery 细节后续读 agent loop/queue：`/code/openclaw/docs/concepts/architecture.md` | `SessionSource` 记录 origin，`build_session_context_prompt` 写入 delivery options，`DeliveryRouter` 通过 adapter `send` 投递到 target：`/code/hermes-agent/gateway/session.py`；`/code/hermes-agent/gateway/delivery.py` | 投递目标应保留为结构化上下文，避免 agent 只凭自然语言猜回复位置。 |

## 源码阅读路线

OpenClaw：

1. `/code/openclaw/docs/concepts/architecture.md`：读概览、组件与流程、连接生命周期、wire protocol、pairing 与 local trust 相关段落。
2. `/code/openclaw/docs/gateway/protocol.md`：读 Transport、Handshake、Framing、Roles + scopes、Caps/commands/permissions。
3. `/code/openclaw/docs/concepts/agent.md`：补上 Gateway 后面的 embedded runtime、workspace、sessions。
4. 后续章节再读 `/code/openclaw/docs/concepts/agent-loop.md` 和 queue/session 相关文档。

Hermes：

1. `/code/hermes-agent/gateway/run.py`：先读 `GatewayRunner.__init__`、`GatewayRunner.start`、`GatewayRunner._handle_message`、`GatewayRunner._handle_message_with_agent`。
2. `/code/hermes-agent/gateway/platforms/base.py`：读 `MessageEvent`、`BasePlatformAdapter`、`BasePlatformAdapter.handle_message`。
3. `/code/hermes-agent/gateway/session.py`：读 `SessionSource`、`SessionContext`、`build_session_context_prompt`、`build_session_key`、`SessionStore.get_or_create_session`、`build_session_context`。
4. `/code/hermes-agent/gateway/delivery.py`：读 `DeliveryRouter`，理解 background/cron/tool delivery 如何回到 adapter。
5. `/code/hermes-agent/run_agent.py`：定位 `AIAgent` 和 `run_conversation`，把 gateway 输入接到 agent loop。

## 最小复现抽象

学习抽象：Gateway 最小复现可以分成六个对象：

1. `GatewayServer`：接收外部连接或平台 webhook/polling update。OpenClaw 用 WebSocket Gateway：`/code/openclaw/docs/gateway/protocol.md`；Hermes 用 platform adapters：`/code/hermes-agent/gateway/platforms/base.py`。
2. `PlatformAdapter`：把外部平台 update、WebSocket frame 或 webhook 归一化为内部事件。Hermes 的现成锚点是 `BasePlatformAdapter` 和 `MessageEvent`：`/code/hermes-agent/gateway/platforms/base.py`；OpenClaw 的协议锚点是 WebSocket req/res/event：`/code/openclaw/docs/gateway/protocol.md`。
3. `SessionRouter`：从 source 生成 session key/session id，并决定这个消息进入哪个 agent/session。Hermes 的现成锚点是 `build_session_key` 和 `SessionStore.get_or_create_session`：`/code/hermes-agent/gateway/session.py`；OpenClaw session store 锚点是 `/code/openclaw/docs/concepts/agent.md`。
4. `DeliveryTarget`：结构化记录回复或后台任务结果应该投递到哪里。Hermes 的锚点是 `SessionSource`、`build_session_context_prompt` 和 `DeliveryRouter`：`/code/hermes-agent/gateway/session.py`；`/code/hermes-agent/gateway/delivery.py`。
5. `AuthPairingStore`：在进入 agent 前验证 token/device/user allowlist，并保存 pairing approval 或 device token。OpenClaw 的锚点是 pairing/auth 文档：`/code/openclaw/docs/concepts/architecture.md`；Hermes 的锚点是 `_is_user_authorized` 与 pairing flow：`/code/hermes-agent/gateway/run.py`。
6. `AgentInvoker`：把 context prompt、history、message 交给 agent loop。Hermes 的锚点是 `_handle_message_with_agent` 调用 `run_conversation`：`/code/hermes-agent/gateway/run.py`；`/code/hermes-agent/run_agent.py`。

## 容易误解的点

- OpenClaw 的 Gateway protocol 是统一 WS RPC；Hermes 的 gateway 不是同一种协议层，而是多平台 adapter runner。这个差异由 OpenClaw protocol 文档和 Hermes `BasePlatformAdapter`/`GatewayRunner` 文档字符串共同支撑：`/code/openclaw/docs/gateway/protocol.md`；`/code/hermes-agent/gateway/platforms/base.py`；`/code/hermes-agent/gateway/run.py`。
- `SessionSource` 不是 transcript 本身。它描述消息来源和投递上下文；session metadata/transcript 由 `SessionStore` 和 `SessionEntry` 管理：`/code/hermes-agent/gateway/session.py`。
- OpenClaw 的 `node` 不是普通聊天用户。Protocol 文档把 `node` 定义为 capability host，需要声明 caps/commands/permissions，且 Gateway server-side enforce allowlists：`/code/openclaw/docs/gateway/protocol.md`。
- Hermes 的 allowlist 不是完整的 sandbox 模型。README 的 Security 文档入口提到 command approval、DM pairing、container isolation，但本章只核实了 gateway run path 中的 allowlist 和 approval bridge：`/code/hermes-agent/README.md`；`/code/hermes-agent/gateway/run.py`。

## 待核实问题

- OpenClaw Gateway 的具体 TypeBox schema、JSON Schema codegen 和 server implementation 尚未展开阅读；本章只依据 docs 描述协议与架构：`/code/openclaw/docs/gateway/protocol.md`。
- Hermes 的各平台 adapter 子类如 Telegram、Discord、Slack、WhatsApp 的实际 auth/polling/webhook 差异尚未逐一核实；本章只核实 `BasePlatformAdapter` 抽象和 `GatewayRunner` 生命周期：`/code/hermes-agent/gateway/platforms/base.py`；`/code/hermes-agent/gateway/run.py`。
- Hermes `DeliveryRouter` 只作为 delivery context 锚点使用，具体 target 解析和 cron delivery 细节应在 automation/session 章节继续核实：`/code/hermes-agent/gateway/delivery.py`。
