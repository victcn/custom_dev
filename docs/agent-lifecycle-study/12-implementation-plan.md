# 类 Codex / Claude Code 工具的最小实现计划

> 参考依据：
> - 本小册子 `01` 到 `11` 章的机制抽象。
> - OpenClaw：`/code/openclaw@ca31a705d02e42ffcfb2c5884bb55339a6d0cbdc`
> - Hermes Agent：`/code/hermes-agent@3d4297a59a8607ed24850524d229f5f42520d087`
> - Codex / Claude Code：官方文档中关于 CLI、SDK、MCP、skills、memory、automation、subagents、hooks 和 failure policy 的产品参照。

## 目标

实现一个类似 Codex / Claude Code 的本地 coding-agent 工具，但吸收 OpenClaw 和 Hermes 的优点：

- 像 Codex / Claude Code 一样：先做好本地 CLI、repo 工作区、工具调用、session continuity、skills、MCP、approval/sandbox。
- 像 OpenClaw 一样：保留 run/session/queue/event 控制面，后续可服务化为 HTTP/SSE/WebSocket gateway。
- 像 Hermes 一样：保留 memory、provider registry、skill 生命周期、automation 和 prompt safety 的扩展点。

第一版不要做完整 OpenClaw Gateway，也不要做 Hermes 式全功能自进化系统。最小可行目标是一个本地开发者工具：

```bash
myagent "修复这个 bug"
myagent chat
myagent run --json "总结这个仓库的测试入口"
myagent resume <session_id>
```

## 推荐路线

推荐采用 **CLI-first 本地 agent host**。

| 路线 | 做法 | 优点 | 缺点 | 结论 |
|---|---|---|---|---|
| CLI-first | 先做本地 CLI、agent loop、tools、session、approval、skills、MCP。 | 最接近 Codex / Claude Code 的核心体验，最快验证闭环。 | 初期没有多平台 Gateway。 | 推荐。 |
| Gateway-first | 先做 daemon、HTTP/SSE/WS、run queue、control plane。 | 长期运行能力强，接近 OpenClaw。 | 第一版复杂度高，容易被控制面拖慢。 | 第二阶段再做。 |
| 单体 AIAgent-first | 先写一个大 `Agent` 类，把 memory/skills/provider/cron 都接进去。 | 启动快。 | 容易变成 Hermes 式巨型主循环，后续边界难拆。 | 只适合原型，不推荐作为长期架构。 |

## 最小架构

```text
CLI / SDK / HTTP API
        |
IngressSurface
        |
RunQueue + SessionStore
        |
AgentLoop
 |      |       |
Prompt  Tools   ModelProvider
 |      |       |
Memory Skills  Transcript/Event Stream
        |
Workspace + Sandbox + Approval
```

核心原则：

- `AgentLoop` 只负责一次 run 的模型/工具闭环，不负责平台路由。
- `SessionStore` 和 `TranscriptStore` 独立，避免把会话身份、消息历史和事件流混在一起。
- `ToolRegistry` 是唯一工具入口，所有工具都要经过 schema、approval、timeout 和审计。
- `MCPBridge` 是外部工具/数据源协议边界，不要塞进 provider registry。
- `ProviderRegistry` 只管模型 provider、auth、model list、fallback。
- `Memory`、`Skills`、`Automation` 都是第二阶段能力，接口先留，功能后补。

## 第一阶段：最小可运行 Coding Agent

目标：做出一个能在本地仓库中完成简单代码任务的 agent。

必须实现：

1. `CLI`
   - `myagent chat`
   - `myagent run "<task>"`
   - `myagent run --json "<task>"`
   - `myagent resume <session_id>`

2. `SessionStore`
   - SQLite 保存 session metadata。
   - JSONL 保存 transcript/events。
   - 每次任务有 `run_id`。
   - 每个会话有 `session_id`。

3. `AgentLoop`
   - 接收 user message。
   - 构造 system prompt。
   - 调用模型。
   - 解析 tool calls。
   - 执行工具。
   - 把 tool results 回填给模型。
   - 产生 final response。
   - 写入 transcript/event log。

4. `ToolRegistry`
   - `list_files`
   - `read_file`
   - `write_file`
   - `edit_file`
   - `apply_patch`
   - `run_command`

5. `ApprovalPolicy`
   - 读文件默认允许。
   - 写文件只允许 workspace 内。
   - shell 命令按风险分级。
   - dangerous command 默认需要确认或拒绝。

6. `WorkspacePolicy`
   - `read-only`
   - `workspace-write`
   - `danger-full-access`

第一阶段不做：

- 多平台聊天接入。
- WebSocket Gateway。
- 自动创建或修改 skills。
- 多 agent task board。
- dreaming/curator。
- 浏览器控制。
- 插件市场。
- 云端执行。

## 第二阶段：Codex / Claude Code 体验层

目标：让它像现代 coding agent，而不是一次性脚本。

实现：

1. repo guidance
   - 支持 `AGENTS.md`。
   - 支持 `.myagent/config.toml`。
   - 支持嵌套 guidance，越靠近 cwd 优先级越高。

2. skills
   - 支持 `.agents/skills/<name>/SKILL.md`。
   - 启动时只加载 `name`、`description`、路径。
   - 相关任务再读完整 `SKILL.md`。
   - 支持 `$skill-name` 显式触发。

3. MCP client
   - 支持 stdio MCP server。
   - 支持 streamable HTTP MCP server。
   - 每个 MCP tool 有 timeout、allow/deny、approval mode。

4. headless / SDK surface
   - `myagent run --json`
   - `myagent serve --stdio`
   - 后续可加 HTTP/SSE。

这一阶段吸收 Codex / Claude Code 的核心优点：CLI、repo guidance、skills、MCP、approval、session continuity。

## 第三阶段：OpenClaw 控制面能力

目标：让 agent 能长期运行、可观测、可取消、可服务化。

实现：

1. `RunQueue`
   - per-session lane：同一 session 串行。
   - global lane：全局并发上限。
   - run cancel / interrupt。
   - stuck run timeout。

2. event stream
   - `lifecycle`
   - `assistant`
   - `tool`
   - `approval`
   - `error`

3. local app-server
   - `POST /runs`
   - `GET /runs/:id`
   - `GET /runs/:id/events` 使用 SSE。
   - `POST /runs/:id/stop`

4. gateway-ready boundary
   - `IngressSurface`
   - `TransportAdapter`
   - `DeliveryTarget`
   - `AuthPairingStore`

这一阶段吸收 OpenClaw 的强项：run/session/queue/event 控制面，而不是直接复制它的完整 Gateway。

## 第四阶段：Hermes 长期能力

目标：让 agent 有长期积累和后台工作能力。

实现：

1. memory
   - `memory.md` 或 SQLite memory。
   - `memory_search`
   - `memory_write`
   - memory 注入必须用 fenced block，避免误当用户输入。

2. provider registry
   - OpenAI。
   - Anthropic。
   - OpenAI-compatible local endpoint。
   - fallback chain。

3. skill lifecycle
   - 记录 skill 使用次数。
   - stale/archive 状态。
   - 第一版只给维护建议，不自动改 skill。

4. automation
   - 简单 scheduled task。
   - local cron-like loop。
   - 每个 automation 使用独立 session 或 worktree。

5. prompt safety
   - 扫描 context files。
   - 扫描 assembled automation prompt。
   - 对外部 skill/MCP instruction 做风险提示。

这一阶段吸收 Hermes 的强项：MemoryManager、ProviderProfile、skills 生命周期、cron prompt safety。

## 模块边界

| 模块 | 职责 | 第一版是否必须 |
|---|---|---|
| `CLI` | 本地交互入口。 | 是 |
| `AgentLoop` | 模型调用、工具调用、tool result 回填、final response。 | 是 |
| `SessionStore` | session id、metadata、resume。 | 是 |
| `TranscriptStore` | JSONL event/transcript。 | 是 |
| `ToolRegistry` | tool schema、dispatch、timeout、approval。 | 是 |
| `WorkspacePolicy` | 文件系统边界。 | 是 |
| `ApprovalPolicy` | 高风险动作确认。 | 是 |
| `ProviderRegistry` | 多 provider 和 fallback。 | 第二阶段末或第四阶段 |
| `SkillRegistry` | skill metadata、progressive disclosure。 | 第二阶段 |
| `MCPBridge` | MCP client/server bridge。 | 第二阶段 |
| `RunQueue` | session lane、global concurrency、cancel。 | 第三阶段 |
| `AppServer` | HTTP/SSE 控制面。 | 第三阶段 |
| `MemoryStore` | 长期事实和偏好。 | 第四阶段 |
| `Scheduler` | 自动化和定时任务。 | 第四阶段 |
| `SubagentRunner` | 子 agent / parallel work。 | 第四阶段之后 |

## 建议技术栈

推荐 TypeScript / Node.js：

- CLI：`commander` 或 `clipanion`
- 配置/schema：`zod`
- 存储：SQLite + JSONL
- 模型：OpenAI Responses API + Anthropic Messages API
- MCP：优先接官方/社区 MCP SDK
- 测试：Vitest
- 格式化：Prettier
- 后续 TUI：Ink

如果团队更偏 Python，也可以用 Python 实现；但 MCP、CLI 分发、Node 生态和前端式 app-server 组合上，TypeScript 更顺手。

## 目录建议

```text
src/
  cli/
    main.ts
    commands/
  core/
    agent-loop.ts
    run-state.ts
    events.ts
  session/
    session-store.ts
    transcript-store.ts
  tools/
    registry.ts
    builtin/
  workspace/
    policy.ts
    approval.ts
  providers/
    registry.ts
    openai.ts
    anthropic.ts
  skills/
    registry.ts
    loader.ts
  mcp/
    bridge.ts
  queue/
    run-queue.ts
  server/
    app-server.ts
  memory/
    memory-store.ts
  automation/
    scheduler.ts
tests/
```

## 里程碑

| 周期 | 目标 | 验收标准 |
|---|---|---|
| Week 1 | CLI + AgentLoop + builtin tools + 单 provider。 | 能在测试仓库读文件、改文件、运行命令，并输出 final response。 |
| Week 2 | SessionStore + TranscriptStore + streaming events + approval。 | 支持 resume；所有 tool 调用有事件记录；危险命令会被拦截或确认。 |
| Week 3 | `AGENTS.md` + skills progressive disclosure + config。 | 能加载 repo guidance；能显式 `$skill`；不会一次性塞入全部 skill 内容。 |
| Week 4 | MCP client + run queue + cancel/timeout。 | 能连接 stdio MCP server；同一 session 串行；run 可取消。 |
| Week 5 | memory + provider fallback。 | 能写入/召回简单 memory；主 provider 失败时可 fallback。 |
| Week 6 | HTTP/SSE app-server + automation prototype。 | 能通过 HTTP 创建 run，通过 SSE 看事件；能跑一个简单定时任务。 |

## 成功标准

第一版成功标准：

- 在本地仓库中完成小型代码修改。
- 修改前能读取上下文，修改后能运行验证命令。
- 所有文件写入和 shell 命令都经过 policy。
- session 可 resume。
- transcript 可审计。
- tool schema 和 tool result 可追踪。
- 支持至少一个 MCP server。
- 支持至少一个 repo guidance 文件。

不以“能接所有平台”“能自动自我进化”“能多 agent 并行写代码”为第一版成功标准。

## 风险与取舍

- 过早做 Gateway 会拖慢核心 coding-agent 闭环。
- 过早做 memory 会污染上下文；memory 必须晚于 transcript 和 prompt boundary。
- 过早做 subagents 会带来并发写冲突；先做 read-heavy parallel work。
- 过早做 plugin market 会带来信任和安全问题；先做本地 skills + MCP。
- provider registry 不要和 MCP 混在一起；前者是模型供应商，后者是工具/数据源协议。

## 最小实现顺序

1. `AgentLoop`
2. `ToolRegistry`
3. `SessionStore`
4. `TranscriptStore`
5. `ApprovalPolicy`
6. `WorkspacePolicy`
7. `ProviderRegistry`
8. `SkillRegistry`
9. `MCPBridge`
10. `RunQueue`
11. `AppServer`
12. `MemoryStore`
13. `Scheduler`
14. `SubagentRunner`

这个顺序的核心是：先闭环，再长期化；先本地安全，再外部扩展；先单 agent 稳定，再多 agent 并行。

