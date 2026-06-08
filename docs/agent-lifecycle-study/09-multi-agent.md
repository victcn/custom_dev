# 多 Agent、委派与任务板

> 参考版本：
> - OpenClaw：`/code/openclaw@ca31a705d02e42ffcfb2c5884bb55339a6d0cbdc`
> - Hermes Agent：`/code/hermes-agent@3d4297a59a8607ed24850524d229f5f42520d087`

## 本章问题

长生命周期 agent 迟早会遇到多身份、多会话、多工作区和多 worker 的问题：同一个 Gateway 如何服务不同 persona，如何把外部消息路由到正确 agent，如何把任务委派给子 agent 或任务板 worker，又如何避免不同 agent 共享错误的凭据、会话和工具权限。本章只写已核实源码或文档事实；`/code/hermes-agent/docs/hermes-kanban-v1-spec.pdf` 未实际解析，所以不把 PDF 内容写成事实。

## OpenClaw 怎么做

OpenClaw 文档声明 Gateway 可以在一个进程中运行多个 isolated agents，每个 agent 有自己的 workspace、state directory `agentDir` 和 session history，inbound messages 通过 bindings 路由到正确 agent。锚点：`/code/openclaw/docs/concepts/multi-agent.md` 的开头与 `What is "one agent"?`。

OpenClaw 的 agent scope 包含 workspace files、auth profiles、model registry 和 session store；文档声明 `agentDir` 下有 per-agent auth profiles，sessions 在 `~/.openclaw/agents/<agentId>/sessions`。锚点：`/code/openclaw/docs/concepts/multi-agent.md` 的 `What is "one agent"?` 与 `Paths`。

OpenClaw 的 routing 规则是 deterministic most-specific wins。文档列出优先级：peer、parentPeer、guildId + roles、guildId、teamId、accountId、channel-level、default agent；同层冲突按 config 顺序，多个 match 字段是 AND 语义。锚点：`/code/openclaw/docs/concepts/multi-agent.md` 的 `Routing rules`。

OpenClaw 的 workspace 是默认 cwd，不是硬 sandbox。文档警告相对路径落在 workspace，但绝对路径仍可访问其他 host 位置，除非启用 sandboxing。锚点：`/code/openclaw/docs/concepts/multi-agent.md` 的 `Workspace note`。

OpenClaw 的 delegate architecture 把 agent 扩展成组织中的 named delegate：它有自己的身份，代表人类行动但不冒充人类，并依赖身份提供方的显式权限。锚点：`/code/openclaw/docs/concepts/delegate-architecture.md` 的 `What is a delegate?`。

OpenClaw delegate 文档强调先 harden 再授权，包括 hard blocks、per-agent tool restrictions、sandbox isolation 和 audit trail；示例中 per-agent `tools.allow`/`tools.deny` 限制 delegate 能调用的工具。锚点：`/code/openclaw/docs/concepts/delegate-architecture.md` 的 `Prerequisites: isolation and hardening`、`Tool restrictions`、`Sandbox isolation`。

OpenClaw subagent 相关 hook 也暴露给插件。`docs/plugins/hooks.md` 列出 `subagent_spawning`、`subagent_delivery_target`、`subagent_spawned`、`subagent_ended`；`docs/tools/index.md` 把 `subagents` 作为内置 session/agent orchestration 工具之一。锚点：`/code/openclaw/docs/plugins/hooks.md`、`/code/openclaw/docs/tools/index.md`。

## Hermes 怎么做

Hermes 的 subagent/delegate 已核实部分集中在 `tools/delegate_tool.py` 和 `run_agent.py`。`delegate_tool.py` 文件注释声明 `delegate_task` 会 spawn child `AIAgent`，子 agent 有 fresh conversation、自己的 `task_id`、受限 toolsets 和由 delegated goal/context 构造的 focused system prompt；父 agent 只看到 delegation call 和 summary result，不看到 child intermediate tool calls 或 reasoning。锚点：`/code/hermes-agent/tools/delegate_tool.py` 文件注释。

Hermes 的 delegate 默认禁止子 agent 使用一组危险或递归工具：`delegate_task`、`clarify`、`memory`、`send_message`、`execute_code`。锚点：`/code/hermes-agent/tools/delegate_tool.py` 的 `DELEGATE_BLOCKED_TOOLS`。

Hermes 的 subagent 并发与深度受配置约束。`delegate_tool.py` 定义 `_DEFAULT_MAX_CONCURRENT_CHILDREN = 3`、`MAX_DEPTH = 1`，并提供 `_get_max_concurrent_children()`、`_get_max_spawn_depth()` 等配置读取路径；`run_agent.py` 的 `_cap_delegate_task_calls()` 会截断超过 `max_concurrent_children` 的 `delegate_task` tool calls。锚点：`/code/hermes-agent/tools/delegate_tool.py`、`/code/hermes-agent/run_agent.py` 的 `_cap_delegate_task_calls()`。

Hermes 的 subagent thread 不继承 CLI 的交互审批回调。`delegate_tool.py` 为 ThreadPoolExecutor worker 安装 `_subagent_auto_deny` 或 `_subagent_auto_approve`，默认自动拒绝危险命令，只有 `delegation.subagent_auto_approve=true` 才自动批准。锚点：`/code/hermes-agent/tools/delegate_tool.py` 的 `_get_subagent_approval_callback()`、`_subagent_auto_deny()`、`_subagent_auto_approve()`。

Hermes 的 parent agent 会追踪 active children 并传播 interrupt。`run_agent.py` 初始化 `_active_children`，`interrupt()` 会遍历 child agent 调用 `child.interrupt()`，`release_clients()` 和 `close()` 会清理 active child agents。锚点：`/code/hermes-agent/run_agent.py` 的 `_active_children` 初始化、`interrupt()`、`release_clients()`、`close()`。

Hermes 的 Kanban guidance 已核实部分在 `agent/prompt_builder.py`。`KANBAN_GUIDANCE` 声明 worker 被分配 `~/.hermes/kanban.db` 中的一个任务，任务 id 在 `$HERMES_KANBAN_TASK`，workspace 在 `$HERMES_KANBAN_WORKSPACE`；worker 应调用 `kanban_show()` 定向、在 workspace 内工作、长任务 heartbeat、需要人类决策时 `kanban_block()`、完成时 `kanban_complete()`、发现后续工作时 `kanban_create()`。锚点：`/code/hermes-agent/agent/prompt_builder.py` 的 `KANBAN_GUIDANCE`。

Hermes 的 Kanban tools 只在 dispatcher worker 或显式 kanban toolset 下暴露。`kanban_tools.py` 注释声明普通 `hermes chat` 没有 kanban tools；`_check_kanban_mode()` 检查 `HERMES_KANBAN_TASK` 或 profile toolsets；`_enforce_worker_task_ownership()` 阻止 worker 修改非自己 task id。锚点：`/code/hermes-agent/tools/kanban_tools.py` 的模块注释、`_check_kanban_mode()`、`_enforce_worker_task_ownership()`。

## 异同点表

| 维度 | OpenClaw | Hermes | 学习结论 |
| --- | --- | --- | --- |
| 多 agent 单位 | 一个 agent 是 workspace、agentDir、auth、model registry、session store 的完整 scope。锚点：`/code/openclaw/docs/concepts/multi-agent.md` | `delegate_task` 子 agent 是一次委派中的 child `AIAgent`，fresh conversation 与独立 task_id。锚点：`/code/hermes-agent/tools/delegate_tool.py` | 多 agent 可以是长期身份，也可以是一次任务的临时 worker。 |
| 路由 | inbound messages 通过 bindings most-specific wins 路由到 agent。锚点：`/code/openclaw/docs/concepts/multi-agent.md` | 已读 Hermes 证据主要是 delegate/kanban worker，不是 OpenClaw 式 channel binding routing。锚点：`/code/hermes-agent/tools/delegate_tool.py`、`/code/hermes-agent/tools/kanban_tools.py` | 外部消息路由和内部任务委派是两类机制，设计时要分开。 |
| 隔离 | per-agent workspace、agentDir、session store；workspace 不是硬 sandbox。锚点：`/code/openclaw/docs/concepts/multi-agent.md` | 子 agent 有独立 task_id 和受限 toolsets；terminal/file 工具对 subagent task_id 有专门处理。锚点：`/code/hermes-agent/tools/delegate_tool.py`、`/code/hermes-agent/run_agent.py` | 隔离至少要覆盖身份、工作区、session 和工具权限，不能只靠 prompt 约定。 |
| 委派架构 | delegate 是有自己身份和组织权限的 agent，不冒充人。锚点：`/code/openclaw/docs/concepts/delegate-architecture.md` | `delegate_task` 是父 agent 在一次 run 内分派短 reasoning/执行子任务的工具。锚点：`/code/hermes-agent/tools/delegate_tool.py` | 委派需要明确“代表谁行动”和“只帮父 agent 做子任务”的区别。 |
| 任务板 | 本章未在 OpenClaw 已读材料中看到 Hermes 式 Kanban DB。锚点：`/code/openclaw/docs/concepts/multi-agent.md` | `KANBAN_GUIDANCE` 和 `kanban_*` tools 以 `~/.hermes/kanban.db` 协调 worker/orchestrator。锚点：`/code/hermes-agent/agent/prompt_builder.py`、`/code/hermes-agent/tools/kanban_tools.py` | 跨 run 的多 worker 协作需要 durable task board，而不是只靠一次上下文。 |
| 插件介入 subagent | plugin hooks 包括 `subagent_spawning` 等。锚点：`/code/openclaw/docs/plugins/hooks.md` | 本章已核实的 Hermes subagent 控制点是 `delegate_tool.py` 与 `run_agent.py`，未确认通用 plugin hook。锚点：`/code/hermes-agent/tools/delegate_tool.py`、`/code/hermes-agent/run_agent.py` | subagent 生命周期如果要被扩展，需要显式 hook 或统一 delegate runner。 |

## 源码阅读路线

1. OpenClaw 先读 `/code/openclaw/docs/concepts/multi-agent.md`，重点看 `What is "one agent"?`、`Paths`、`Routing rules`、workspace note。
2. OpenClaw 再读 `/code/openclaw/docs/concepts/delegate-architecture.md`，重点看 `What is a delegate?`、capability tiers、hardening、tool restrictions、sandbox isolation。
3. OpenClaw subagent extension 读 `/code/openclaw/docs/plugins/hooks.md` 的 `Subagents` hook catalog，以及 `/code/openclaw/docs/tools/index.md` 的 `subagents` 工具条目。
4. Hermes 先读 `/code/hermes-agent/tools/delegate_tool.py` 的文件注释、`DELEGATE_BLOCKED_TOOLS`、approval callback、active subagent registry。
5. Hermes 再读 `/code/hermes-agent/run_agent.py` 的 `_active_children`、`interrupt()`、`_cap_delegate_task_calls()`、`_dispatch_delegate_task()`。
6. Hermes Kanban 读 `/code/hermes-agent/agent/prompt_builder.py` 的 `KANBAN_GUIDANCE`，再读 `/code/hermes-agent/tools/kanban_tools.py` 的 `_check_kanban_mode()`、`_enforce_worker_task_ownership()`。
7. Hermes PDF `/code/hermes-agent/docs/hermes-kanban-v1-spec.pdf` 待实际解析后再加入事实。

## 最小复现抽象

这是学习抽象，不是两个项目的精确实现。

- `AgentProfile`：描述 agent 的身份、workspace、state dir、auth store、tool policy、model policy；对应 OpenClaw multi-agent agent scope 与 Hermes child agent 配置。
- `WorkspaceAllocator`：为 agent 或 worker 分配默认 cwd、task workspace 和可选 sandbox；对应 OpenClaw workspace/agentDir 与 Hermes `$HERMES_KANBAN_WORKSPACE`。
- `DelegationQueue`：父 agent 提交 child task，限制并发、深度、工具集和审批策略；对应 Hermes `delegate_task` 与 OpenClaw subagent hooks。
- `TaskBoard`：持久任务、状态、父子依赖、heartbeat、block、complete、handoff；对应 Hermes `~/.hermes/kanban.db` 与 `kanban_*` tools。
- `SubagentRunner`：启动 fresh child run，隔离上下文，收集 summary，向父 agent 或任务板交付结果；对应 Hermes `delegate_tool.py` 和 OpenClaw subagent/tool docs。

## 容易误解的点

- OpenClaw 的 multi-agent 不是只给同一个模型取多个名字；文档把 workspace、auth、model registry、session store 都纳入 agent scope。锚点：`/code/openclaw/docs/concepts/multi-agent.md`。
- OpenClaw workspace 隔离不是安全沙箱；文档明确绝对路径仍可能访问 host 其他位置，除非启用 sandbox。锚点：`/code/openclaw/docs/concepts/multi-agent.md`。
- Hermes `delegate_task` 不是 Kanban board 的替代品。`KANBAN_GUIDANCE` 明确说 board tasks 用 `kanban_*`，`delegate_task` 用于自己 run 内的短 reasoning subtasks。锚点：`/code/hermes-agent/agent/prompt_builder.py`。
- Hermes worker 不能随意改其他 task。`_enforce_worker_task_ownership()` 会在 dispatcher-scoped worker 中拒绝修改非 `$HERMES_KANBAN_TASK` 的 task。锚点：`/code/hermes-agent/tools/kanban_tools.py`。

## 待核实问题

- 尚未解析 `/code/hermes-agent/docs/hermes-kanban-v1-spec.pdf`，因此不能把 PDF 中可能存在的 Kanban v1 设计写成本章事实。
- OpenClaw subagent runtime 源码路径尚未系统阅读，本章只核实到 docs 和 plugin hook catalog。
- Hermes gateway/platform 层是否有 OpenClaw 式 multi-agent channel binding routing，需要继续读 `/code/hermes-agent/gateway/*`。
