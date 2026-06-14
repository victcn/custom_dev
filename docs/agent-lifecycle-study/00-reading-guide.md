# 阅读指南

> 参考版本：
> - OpenClaw：`/code/openclaw@ca31a705d02e42ffcfb2c5884bb55339a6d0cbdc`
> - Hermes Agent：`/code/hermes-agent@3d4297a59a8607ed24850524d229f5f42520d087`
> - Codex / Claude Code：作为入口协议、CLI/SDK/MCP/tool-use、memory、skills、automation、subagents、hooks/failure policy 等机制的官方文档参照，不作为本小册子的逐章源码对照对象。

## 本章问题

这套小册子不是 API 手册，而是“机制优先”的源码阅读材料：先问一个长生命周期 agent 机制解决什么问题，再看 OpenClaw 和 Hermes Agent 各自怎样落地，最后抽象出可复现的最小模型。涉及入口、session、loop、memory、skills、automation、tools/MCP、subagents 或 failure policy 时，本小册子也会把 Codex 和 Claude Code 纳入官方文档参照，用来避免把所有 coding agent 都误归类为 OpenClaw 式 Gateway，或把 CLI/SDK/MCP/tool-use 这类 host surface 忽略掉。这个写作目标、章节边界和证据规则来自本目录的设计规格：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/design.md`。

阅读时要区分三类表述：

- 源码事实：直接由代码结构、类、函数或配置读取出来，例如 Hermes 的 `SessionSource`、`SessionContext` 和 `SessionStore` 位于 `/code/hermes-agent/gateway/session.py`。
- 文档声明：项目 README 或 docs 明确说明的定位、能力或默认行为，例如 OpenClaw README 将 OpenClaw 定位为运行在用户设备上的 personal AI assistant，并列出 Gateway、channels、tools、skills 和 security defaults：`/code/openclaw/README.md`。
- 官方文档参照：Codex 和 Claude Code 这类未纳入本地源码阅读的对象，只使用官方公开文档说明其入口、接口和协议边界，例如 Codex CLI 与 Claude Agent SDK/MCP 文档。
- 学习抽象：为了学习和最小复现而做的归纳，例如“入口、会话、记忆、自动化、工具边界共同构成长生命周期 agent 的骨架”；它不是任一项目的逐字实现，依据来自本小册子设计规格 `/code/code-dev/agent_dev/docs/agent-lifecycle-study/design.md`。

## OpenClaw 怎么做

OpenClaw 的文档声明是“personal AI assistant”，强调运行在自己的设备上，并把 Gateway 称为控制面而非产品本身；README 还说明推荐通过 `openclaw onboard --install-daemon` 安装 Gateway daemon，并通过 `openclaw gateway`、`openclaw agent` 等命令交互：`/code/openclaw/README.md`。

OpenClaw 的入口阅读应先看 Gateway，再看 agent runtime。Gateway architecture 文档说明单个长驻 Gateway 拥有消息 surface，CLI、web UI、macOS app 和 automations 通过 WebSocket 接入，nodes 也通过同一 WebSocket server 接入但声明 `role: node`：`/code/openclaw/docs/concepts/architecture.md`。Agent runtime 文档说明 OpenClaw 运行 single embedded agent runtime，使用 `agents.defaults.workspace` 作为工具和上下文的工作目录，并将 session transcript 存为 JSONL：`/code/openclaw/docs/concepts/agent.md`。

## Hermes 怎么做

Hermes 的文档声明是 self-improving AI agent，README 说明它有内置 learning loop、memory、skills、cron、subagents，并支持 CLI 与 messaging gateway 两类入口：`/code/hermes-agent/README.md`；中文 README 也用中文说明“自进化 AI 代理”“闭环学习”“定时自动化”和“委派与并行”：`/code/hermes-agent/README.zh-CN.md`。

Hermes 的入口阅读应从 `run_agent.py` 与 `gateway/` 并行开始。`run_agent.py` 中的 `AIAgent` 和 `run_conversation` 是主 agent loop 的核心锚点：`/code/hermes-agent/run_agent.py`。Messaging gateway 的 session 来源、会话上下文和动态 system prompt 注入在 `SessionSource`、`SessionContext`、`build_session_context_prompt`、`SessionStore` 中：`/code/hermes-agent/gateway/session.py`。Provider 层用 `ProviderProfile` 和 registry 统一 auth、transport kwargs、model listing 与 runtime routing：`/code/hermes-agent/providers/README.md`。

## 异同点表

| 维度 | OpenClaw | Hermes Agent |
|---|---|---|
| 项目定位 | 文档声明为 local-first personal AI assistant，Gateway 是控制面：`/code/openclaw/README.md` | 文档声明为 self-improving AI agent，强调 learning loop、skills、memory、cron、subagents：`/code/hermes-agent/README.md` |
| 推荐入口 | `openclaw onboard --install-daemon`、`openclaw gateway`、`openclaw agent`：`/code/openclaw/README.md` | `hermes`、`hermes gateway`、`hermes setup`：`/code/hermes-agent/README.md` |
| Gateway 阅读起点 | `/code/openclaw/docs/concepts/architecture.md` 和 `/code/openclaw/docs/gateway/protocol.md` | `/code/hermes-agent/gateway/run.py`、`/code/hermes-agent/gateway/session.py`、`/code/hermes-agent/gateway/platforms/base.py` |
| Agent runtime 阅读起点 | `/code/openclaw/docs/concepts/agent.md` | `/code/hermes-agent/run_agent.py` |
| Provider/插件阅读起点 | OpenClaw README 指向 plugins、tools、skills 和 model configuration：`/code/openclaw/README.md` | Hermes provider registry 明确集中在 `providers/README.md`，profiles 来自 `plugins/model-providers/<name>/` 和 `$HERMES_HOME/plugins/model-providers/<name>/`：`/code/hermes-agent/providers/README.md` |

## 源码阅读路线

推荐顺序：

1. 先读 `01-architecture-overview.md`，建立入口、状态、记忆、技能、自动化和 provider/plugin 的总图；章节依据包括 `/code/openclaw/README.md`、`/code/openclaw/docs/concepts/architecture.md`、`/code/openclaw/docs/concepts/agent.md`、`/code/hermes-agent/README.md`、`/code/hermes-agent/providers/README.md`。
2. 再读 `02-gateway.md`，理解外部消息如何进入长驻进程、HTTP/API/Webhook/CLI/SDK/MCP 入口如何成为 session 或 run，以及如何再交给 agent loop；章节依据包括 `/code/openclaw/docs/gateway/protocol.md`、`/code/hermes-agent/gateway/run.py`、`/code/hermes-agent/gateway/session.py`，并补充 Codex CLI 与 Claude Agent SDK/MCP 官方文档参照。
3. 然后读后续 `03-agent-loop.md`，把 Gateway 与 LLM/tool loop 接上；初始锚点是 `/code/openclaw/docs/concepts/agent-loop.md` 和 `/code/hermes-agent/run_agent.py`。
4. 按需求跳读 `05-memory.md`、`06-skills.md`、`07-automation.md`：OpenClaw 起点分别是 `/code/openclaw/docs/concepts/memory.md`、`/code/openclaw/docs/tools/skills.md`、`/code/openclaw/docs/automation/cron-jobs.md`；Hermes 起点分别是 `/code/hermes-agent/agent/memory_manager.py`、`/code/hermes-agent/agent/curator.py`、`/code/hermes-agent/cron/scheduler.py`。
5. 如果目标是自己实现一个类似 Codex / Claude Code 的工具，读完 `11-minimal-reproduction.md` 后继续读 `12-implementation-plan.md`；它把 OpenClaw/Hermes 的源码学习结论和 Codex/Claude Code 的官方参照收敛成一个最小实现路线。

术语表：

| 术语 | 本小册子中的用法 | 起始锚点 |
|---|---|---|
| `Gateway` | 长驻入口/控制面，把平台消息、控制命令、路由和投递接到 agent | `/code/openclaw/docs/concepts/architecture.md`；`/code/hermes-agent/gateway/run.py` |
| `MCP` | agent host 和外部工具/数据源之间的标准化扩展协议；在 Codex 和 Claude Code 中是重要集成面 | Codex CLI configuration/reference；Claude MCP connector 官方文档 |
| `agent loop` | 组装上下文、调用模型、执行工具、生成输出、持久化状态的循环 | `/code/openclaw/docs/concepts/agent.md`；`/code/hermes-agent/run_agent.py` |
| `session` | 稳定会话身份与 transcript/metadata 的组合 | `/code/openclaw/docs/concepts/agent.md`；`/code/hermes-agent/gateway/session.py` |
| `memory` | 跨轮次或跨会话保留并召回的信息 | `/code/openclaw/docs/concepts/agent.md`；`/code/hermes-agent/README.md` |
| `skill` | 可被发现、加载、复用的过程记忆或操作说明 | `/code/openclaw/docs/concepts/agent.md`；`/code/hermes-agent/README.md` |
| `tool` | agent 可调用的外部能力边界，例如文件、终端、平台发送、浏览器 | `/code/openclaw/README.md`；`/code/hermes-agent/README.md` |
| `cron` | 后台定时任务或自然语言调度入口 | `/code/openclaw/README.md`；`/code/hermes-agent/README.md` |
| `subagent` | 用于委派或并行工作流的独立 agent 执行单元 | `/code/hermes-agent/README.md`；OpenClaw README 的 multi-agent routing 入口见 `/code/openclaw/README.md` |

## 最小复现抽象

学习抽象：读这套材料时，可以把长生命周期 agent 看成五层：入口层 `Gateway`、会话层 `session`、执行层 `agent loop`、长期能力层 `memory`/`skill`、后台层 `cron`/automation。这个分层是本小册子的学习组织方式，不是 OpenClaw 或 Hermes 的源码目录划分；真实项目锚点分别见 `/code/openclaw/docs/concepts/architecture.md`、`/code/openclaw/docs/concepts/agent.md`、`/code/hermes-agent/gateway/run.py`、`/code/hermes-agent/run_agent.py`。

最小复现时不要先追求所有平台。先实现一个本地输入入口、一个 session key 生成器、一个 transcript store、一个 agent loop stub、一个可插拔 tool registry、一个后台定时触发器。这样能覆盖 OpenClaw README 中的 Gateway/session/tools/skills/cron 线索和 Hermes README 中的 CLI/gateway/memory/skills/cron 线索：`/code/openclaw/README.md`；`/code/hermes-agent/README.md`。

## 容易误解的点

- 不要把“Gateway”理解成“agent 本体”。OpenClaw README 明确说 Gateway 是 control plane，product 是 assistant：`/code/openclaw/README.md`；Hermes `GatewayRunner` 文档字符串说它管理 platform adapters 生命周期并把消息路由到/来自 agent：`/code/hermes-agent/gateway/run.py`。
- 不要把“中文正文”误解为翻译所有符号名。文件路径、函数名、类名、命令和配置键必须保留原文；这是本目录设计规格的语言要求：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/design.md`。
- 不要把“学习抽象”当成源码事实。比如五层模型只是学习路线，真实 OpenClaw 的 embedded runtime 和 Hermes 的 `AIAgent`/`GatewayRunner` 分布在不同文件：`/code/openclaw/docs/concepts/agent.md`；`/code/hermes-agent/gateway/run.py`；`/code/hermes-agent/run_agent.py`。

## 待核实问题

- OpenClaw 的 provider/plugin 细节还需要在后续 `08-tools-plugins.md` 继续核实，当前章节只引用 README 和 architecture/agent 文档：`/code/openclaw/README.md`。
- Hermes 中文 README 与英文 README 在 Windows 支持描述上可能存在版本差异，本小册子后续如涉及安装兼容性，应回到 `/code/hermes-agent/README.md` 与 `/code/hermes-agent/README.zh-CN.md` 对照核实。
