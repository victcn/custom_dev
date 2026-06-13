# 失败模式与恢复策略

> 参考版本：
> - OpenClaw：`/code/openclaw@ca31a705d02e42ffcfb2c5884bb55339a6d0cbdc`
> - Hermes Agent：`/code/hermes-agent@3d4297a59a8607ed24850524d229f5f42520d087`
> - Codex：OpenAI Codex manual，核验于 2026-06-13：`https://developers.openai.com/codex/codex-manual.md`
> - Claude Code：Anthropic hooks/security/scheduled tasks 官方文档，核验于 2026-06-13：`https://code.claude.com/docs/en/hooks`；`https://code.claude.com/docs/en/scheduled-tasks`

## 本章问题

长生命周期 agent 的失败不是单点异常，而是“运行太久、外部依赖不稳定、工具权限过大、上下文被污染、队列被卡住”叠加后的系统问题。学习本章时可以把失败处理拆成六类：超时、重试、可取消执行、卡住诊断、沙箱/权限边界、prompt injection 扫描。这个分类是学习抽象，依据来自 OpenClaw 的 agent loop、queue、retry、sandboxing、安全文档，以及 Hermes 的 `run_agent.py` 和 `cron/scheduler.py`。锚点：`/code/openclaw/docs/concepts/agent-loop.md`、`/code/openclaw/docs/concepts/queue.md`、`/code/openclaw/docs/concepts/retry.md`、`/code/openclaw/docs/gateway/sandboxing.md`、`/code/openclaw/docs/gateway/security/index.md`、`/code/hermes-agent/run_agent.py`、`/code/hermes-agent/cron/scheduler.py`。

## OpenClaw 怎么做

OpenClaw 把 agent run 放在 Gateway 管理的 agent loop 内，`agent` RPC 接受任务后立即返回 `{ runId, acceptedAt }`，实际运行由 `agentCommand` 和 `runEmbeddedPiAgent` 推进；`agent.wait` 等待指定 `runId` 的 lifecycle end/error，但 `agent.wait` 自身的超时只是等待超时，不等同于停止 agent。锚点：`/code/openclaw/docs/concepts/agent-loop.md`。

OpenClaw 有多层 timeout。`agent.wait` 默认 30 秒且可由 `timeoutMs` 覆盖；agent runtime 的 `agents.defaults.timeoutSeconds` 默认 172800 秒，并在 `runEmbeddedPiAgent` 中通过 abort timer 执行；cron-triggered run 的外层 `timeoutSeconds` 由 cron 拥有。锚点：`/code/openclaw/docs/concepts/agent-loop.md`。

OpenClaw 还区分 model idle timeout 和 provider HTTP request timeout：`models.providers.<id>.timeoutSeconds` 可扩展慢速本地或自托管 provider 的模型请求、SDK request timeout、guarded fetch abort 和 stream idle watchdog；无显式 model 或 agent timeout 的 cron run 会禁用 idle watchdog，依赖 cron 外层 timeout。锚点：`/code/openclaw/docs/concepts/agent-loop.md`。

OpenClaw 对 stuck session 有诊断分类：长时间 `processing` 且没有 reply/tool/status/block/ACP progress 的 session 会按当前活动区分为 `session.long_running`、`session.stalled` 或 `session.stuck`；只有 stale session bookkeeping 路径会立即释放 session lane，stalled embedded run 要等 `diagnostics.stuckSessionAbortMs` 后 abort-drain。锚点：`/code/openclaw/docs/concepts/agent-loop.md`、`/code/openclaw/docs/concepts/queue.md`。

OpenClaw 的 retry policy 明确“按 HTTP request 重试，而不是按 multi-step flow 重试”，默认 attempts 为 3，max delay cap 为 30000 ms，jitter 为 0.1；Discord/Telegram 会对 rate limit、timeout、5xx 和 transient transport failures 做重试，但 Markdown parse errors 不重试而降级为 plain text。锚点：`/code/openclaw/docs/concepts/retry.md`。

OpenClaw 对 provider SDK 的长 `Retry-After` 有特殊处理：Stainless-based SDK 遇到 `408`、`409`、`429` 和 `5xx` 且 wait 超过 60 秒时，OpenClaw 注入 `x-should-retry: false`，让错误尽快浮出并触发 model failover；可用 `OPENCLAW_SDK_RETRY_MAX_WAIT_SECONDS` 覆盖。锚点：`/code/openclaw/docs/concepts/retry.md`。

OpenClaw 的 sandboxing 是可选的，配置键是 `agents.defaults.sandbox` 或 `agents.list[].sandbox`；开启后 tool execution 可进入 sandbox backend，Gateway process 本身不在 sandbox 内。锚点：`/code/openclaw/docs/gateway/sandboxing.md`。

OpenClaw 的 sandbox mode 包括 `off`、`non-main`、`all`，scope 包括 `agent`、`session`、`shared`，backend 包括 `docker`、`ssh`、`openshell`；文档明确说明 sandbox 不是完美安全边界，而是降低 filesystem/process blast radius。锚点：`/code/openclaw/docs/gateway/sandboxing.md`。

OpenClaw 安全文档把默认威胁模型限定为 single-user personal assistant trust model，不把一个共享 Gateway/agent 描述成 hostile multi-tenant security boundary；文档也给出 hardened baseline，例如 `gateway.bind: "loopback"`、`tools.exec.security: "deny"`、`tools.exec.ask: "always"`、`tools.elevated.enabled: false`。锚点：`/code/openclaw/docs/gateway/security/index.md`。

## Hermes 怎么做

Hermes 的主循环在 `AIAgent.run_conversation` 周边组织，初始化阶段会调用 `_install_safe_stdio()`，把 `stdout`/`stderr` 包装成 `_SafeWriter`，从而捕获 broken pipe、closed file 等 `OSError`/`ValueError`，避免 daemon/headless 场景中一次 print 失败打断 agent setup 或 `run_conversation()`。锚点：`/code/hermes-agent/run_agent.py::_SafeWriter`、`/code/hermes-agent/run_agent.py::_install_safe_stdio`。

Hermes 对 tool call 参数有修复函数 `_repair_tool_call_arguments`：空参数会转为 `{}`，Python `None` 会规范化为 `{}`，JSON 字符串中的控制字符、尾逗号、缺失闭合括号等会尝试修复；不可修复时用 `{}` 兜底，避免 session 因 malformed tool_call arguments 崩掉。锚点：`/code/hermes-agent/run_agent.py::_repair_tool_call_arguments`。

Hermes 在处理 assistant tool calls 时会先修复不匹配的工具名，再校验是否属于 `valid_tool_names`；无效工具名会作为 tool result 反馈给模型让其自我修正，累计 3 次后返回 `partial`。锚点：`/code/hermes-agent/run_agent.py` 中 “Validate tool call names - detect model hallucinations” 分支。

Hermes 对 invalid JSON tool arguments 先短路重试 API call，达到 3 次后把 assistant message 和 tool error result 注入消息历史，让模型用 tool role 结果恢复；如果判断为 truncated tool call arguments，则拒绝执行并以 `partial` 返回。锚点：`/code/hermes-agent/run_agent.py` 中 “Validate tool call arguments are valid JSON” 分支。

Hermes 的 API retry 上限来自 `agent.api_max_retries`，默认 3，且最小为 1；失败处理会记录 provider、model、endpoint、上下文规模等诊断信息，并在 rate limit 时优先读取 `Retry-After`，否则使用 `jittered_backoff`。锚点：`/code/hermes-agent/run_agent.py` 初始化 `self._api_max_retries` 的分支、`/code/hermes-agent/run_agent.py` API error recovery 分支。

Hermes 有 provider fallback 机制：初始化阶段保存 `_fallback_chain`，错误处理里在 rate limit、non-retryable error 或 max retries exhausted 等路径尝试 `_try_activate_fallback()`。锚点：`/code/hermes-agent/run_agent.py` 中 “Provider fallback chain” 与 “trying fallback” 分支、`/code/hermes-agent/providers/README.md`。

Hermes 的 stream retry 诊断会记录 provider、base_url、subagent_id、HTTP status、streamed bytes/chunks、elapsed、upstream headers 等信息，并给用户显示简短的 reconnecting/retry 状态。锚点：`/code/hermes-agent/run_agent.py::_log_stream_retry`、`/code/hermes-agent/run_agent.py::_emit_stream_drop`。

Hermes 的 cron scheduler 在 `_build_effective_prompt` 中先拼接 cron guidance、script output、`context_from` output 和 skill 内容，然后调用 `_scan_assembled_cron_prompt`；扫描命中时抛出 `CronPromptInjectionBlocked`，`run_job` 会拒绝运行 agent 并给 operator 清晰错误。锚点：`/code/hermes-agent/cron/scheduler.py::_build_effective_prompt`、`/code/hermes-agent/cron/scheduler.py::_scan_assembled_cron_prompt`、`/code/hermes-agent/cron/scheduler.py::CronPromptInjectionBlocked`。

Hermes 的 cron timeout 是 inactivity-based timeout：默认读取 `HERMES_CRON_TIMEOUT`，未配置时 600 秒，`0` 表示 unlimited；scheduler 把 `agent.run_conversation(prompt)` 放入 worker thread，每 5 秒检查 `agent.get_activity_summary()`，超过 idle limit 后调用 `agent.interrupt("Cron job timed out (inactivity)")` 并抛出 `TimeoutError`。锚点：`/code/hermes-agent/cron/scheduler.py` 中 “Run the agent with an inactivity-based timeout” 分支。

## Codex / Claude Code 参考

Codex manual 把失败边界集中在 sandbox、approval、MCP trust、hooks、rules 和 CI/automation safety 上。Codex sandbox 有 read-only、workspace-write、full access 等 presets；automations 在后台运行且使用默认 sandbox，full access 会提高风险；GitHub Action 文档强调限制触发者、保护 API key、避免把未清洗的 PR/issue 内容直接喂给 Codex（官方文档：`https://developers.openai.com/codex/codex-manual.md` 的 `Agent approvals & security`、`Sandbox`、`Automations`、`Codex GitHub Action`）。

Codex MCP 配置也有失败面：stdio/streamable HTTP server 有 startup timeout、tool timeout、required flag、enabled/disabled tools、default tools approval mode 和 per-tool override。若 MCP server 不可用，`required = true` 可使 startup fail；若工具权限过宽，外部 tool action 会扩大 blast radius（官方文档：`https://developers.openai.com/codex/codex-manual.md` 的 `Model Context Protocol`）。

Claude Code 的 hooks 文档把 `PreToolUse`、`PermissionRequest`、`PostToolUse`、`UserPromptSubmit`、`Stop`、`SubagentStart`、`SubagentStop` 等事件作为 lifecycle control points；hooks 可以阻止工具调用、记录审计、扫描 prompt 或在 turn 停止时执行检查。Scheduled tasks 还有明确的失败/控制边界：低优先级入队、当前 turn 结束后触发、50 个 task 上限、7 天 recurring expiry、`CLAUDE_CODE_DISABLE_CRON=1` 可关闭 scheduler（官方文档：`https://code.claude.com/docs/en/hooks`；`https://code.claude.com/docs/en/scheduled-tasks`）。

因此加入 Codex/Claude Code 后，本章的恢复策略要多一层“host policy failure”：即使 agent loop 没崩，也可能因为 sandbox 拒绝、approval policy、hook denial、MCP startup/tool timeout、subagent hook、scheduled task expiry 或 CI trigger policy 而停止。最小实现应把这些停止原因结构化记录，不要只归为 model error 或 tool exception。

## 异同点表

| 维度 | OpenClaw | Hermes | 学习结论 |
|---|---|---|---|
| agent timeout | `agents.defaults.timeoutSeconds` 默认 48 小时，由 `runEmbeddedPiAgent` abort timer 执行。锚点：`/code/openclaw/docs/concepts/agent-loop.md` | cron 使用 inactivity timeout；主 `AIAgent` 还通过 retry/interrupt/iteration budget 控制失败面。锚点：`/code/hermes-agent/cron/scheduler.py`、`/code/hermes-agent/run_agent.py::IterationBudget` | 长生命周期系统至少要区分“总运行期限”和“无活动期限”。 |
| wait timeout | `agent.wait` 默认 30 秒，只影响等待结果。锚点：`/code/openclaw/docs/concepts/agent-loop.md` | 本章未核实 Hermes 是否有等价的 `agent.wait` RPC。 | 等待超时不能被误读为取消执行。 |
| model idle timeout | `models.providers.<id>.timeoutSeconds` 作用于 provider HTTP fetch、SDK request timeout 和 stream idle watchdog。锚点：`/code/openclaw/docs/concepts/agent-loop.md` | cron 根据 `agent.get_activity_summary()` 判断 inactivity；stream drop 有 retry 诊断。锚点：`/code/hermes-agent/cron/scheduler.py`、`/code/hermes-agent/run_agent.py::_emit_stream_drop` | idle watchdog 需要与外层 runtime timeout 分离。 |
| retry policy | 明确按单个 HTTP request 重试，默认 attempts 3、max delay 30000 ms、jitter 0.1。锚点：`/code/openclaw/docs/concepts/retry.md` | `agent.api_max_retries` 默认 3，错误分支结合 `Retry-After`、`jittered_backoff`、fallback provider。锚点：`/code/hermes-agent/run_agent.py` | 重试要限定粒度，避免重复非幂等多步骤动作。 |
| stuck session | 诊断分 `session.long_running`、`session.stalled`、`session.stuck`，并控制 lane release/abort-drain。锚点：`/code/openclaw/docs/concepts/agent-loop.md`、`/code/openclaw/docs/concepts/queue.md` | 本章只核实 cron inactivity timeout 和 agent activity summary，未核实是否有 session lane stuck 分类。锚点：`/code/hermes-agent/cron/scheduler.py` | 卡住恢复需要“活动分类”，不能只用一个 timeout。 |
| tool call repair | 本章未核实 OpenClaw 是否有 Hermes 等价的 tool call 参数修复层。 | 修复工具名、修复/反馈 invalid JSON arguments，3 次阈值后返回 partial 或注入 tool error results。锚点：`/code/hermes-agent/run_agent.py` | 工具调用恢复可以先让模型自我修正，再降级为 partial。 |
| prompt injection scanning | 安全文档强调 prompt/content guardrails 只是降低风险，不是 auth/sandbox boundary。锚点：`/code/openclaw/docs/gateway/security/index.md` | cron 对组装后的 prompt 扫描，包括 runtime 加载的 skill 内容。锚点：`/code/hermes-agent/cron/scheduler.py::_scan_assembled_cron_prompt` | 自动化任务必须扫描最终送入模型的 prompt，而不只是创建时的用户字段。 |
| sandboxing | 可选，mode/scope/backend 可配置，Gateway 不进 sandbox。锚点：`/code/openclaw/docs/gateway/sandboxing.md` | README 声明有 local/Docker/SSH/Singularity/Modal/Daytona/Vercel Sandbox terminal backends；本章未核实其安全边界实现。锚点：`/code/hermes-agent/README.md` | 沙箱是 blast radius 控制，不应被写成绝对安全保证。 |

## Codex / Claude Code 参照表

| 维度 | Codex | Claude Code | 学习结论 |
|---|---|---|---|
| sandbox/approval | read-only、workspace-write、full access、approval policy、rules/admin requirements 共同限制动作。 | permissions/hooks 可拦截工具、prompt、stop、subagent 生命周期。 | host policy failure 要独立于 model/tool exception 记录。 |
| MCP 失败 | MCP server 有 startup timeout、tool timeout、required、enabled/disabled tools。 | remote MCP/API tool use 也会引入外部服务失败和权限边界。 | MCP failure 应分为连接失败、工具超时、权限拒绝和结果错误。 |
| unattended run | automations/GitHub Action 需要触发者控制、secret 保护和 prompt injection 清洗。 | scheduled tasks 有低优先级入队、task 上限、过期和禁用开关。 | 后台任务失败不一定是 agent loop 失败，可能是 scheduler/CI/policy 结果。 |
| lifecycle hooks | hooks 可在 tool、compact、prompt、stop、subagent 事件运行。 | hooks 是阻止动作、审计和恢复的关键控制点。 | 最小恢复模型应保留 hook denial、timeout 和 retryable 分类。 |

## 源码阅读路线

1. 先读 `/code/openclaw/docs/concepts/agent-loop.md` 的 `Timeouts` 与 `Where things can end early`，建立 runtime timeout、wait timeout、model idle timeout 的边界。
2. 读 `/code/openclaw/docs/concepts/queue.md` 的 `Troubleshooting`，理解 stuck/stalled/long-running 诊断如何避免 session lane 永久阻塞。
3. 读 `/code/openclaw/docs/concepts/retry.md`，确认 retry 粒度、默认值、provider SDK 长等待处理。
4. 读 `/code/openclaw/docs/gateway/sandboxing.md` 的 `What gets sandboxed`、`Modes`、`Scope`、`Backend`，再读 `/code/openclaw/docs/gateway/security/index.md` 的 trust model 和 hardened baseline。
5. 读 `/code/hermes-agent/run_agent.py::_SafeWriter` 和 `_install_safe_stdio`，理解 safe writer 解决的是 daemon/headless stdio 失败。
6. 读 `/code/hermes-agent/run_agent.py::_repair_tool_call_arguments`，再跳到 `AIAgent.run_conversation` 中 tool call validation 分支。
7. 读 `/code/hermes-agent/run_agent.py` 中 `self._api_max_retries`、`_log_stream_retry`、`_emit_stream_drop`、API error recovery 和 fallback 分支。
8. 读 `/code/hermes-agent/cron/scheduler.py::_build_effective_prompt`、`_scan_assembled_cron_prompt`、cron inactivity timeout 分支。
9. Codex 读 manual 的 `Agent approvals & security`、`Sandbox`、`Model Context Protocol`、`Hooks`、`Automations`、`Codex GitHub Action`，确认 host policy、MCP timeout 和 unattended run 的失败边界。
10. Claude Code 读 `https://code.claude.com/docs/en/hooks` 与 `https://code.claude.com/docs/en/scheduled-tasks`，确认 hook denial、scheduled task 入队/上限/过期和禁用开关。

## 最小复现抽象

以下是学习抽象，不代表 OpenClaw 或 Hermes 的精确实现。

```python
class TimeoutPolicy:
    runtime_seconds: int | None
    idle_seconds: int | None
    wait_seconds: int | None

class RetryPolicy:
    attempts: int
    min_delay_ms: int
    max_delay_ms: int
    jitter: float
    retry_after_cap_seconds: int | None

class AbortController:
    def abort(self, reason: str) -> None: ...
    def signal(self): ...

class SandboxPolicy:
    mode: str        # off | non-main | all
    scope: str       # agent | session | shared
    backend: str     # docker | ssh | local | remote
    workspace_access: str

class PromptInjectionScanner:
    def scan_final_prompt(self, text: str) -> None: ...

class RecoveryMonitor:
    def touch(self, kind: str, detail: str) -> None: ...
    def classify(self) -> str: ...  # long_running | stalled | stuck

class HostPolicyFailure:
    source: str      # sandbox | approval | hook | mcp | ci | scheduler
    reason: str
    retryable: bool
```

最小流程是：入口创建 `AbortController`；`AgentLoop` 同时安装 runtime timer 和 idle watchdog；每次模型 chunk、tool event、status event 调用 `RecoveryMonitor.touch()`；provider call 按 `RetryPolicy` 重试；自动化任务在“最终组装 prompt”后调用 `PromptInjectionScanner.scan_final_prompt()`；工具执行前根据 `SandboxPolicy` 决定 workspace/backend；MCP server startup/tool timeout、hook denial、approval denial、CI trigger policy 和 scheduler expiry 记录为 `HostPolicyFailure`；诊断线程或队列调度器按 `RecoveryMonitor.classify()` 决定记录、等待、abort-drain 或释放 lane。抽象依据：`/code/openclaw/docs/concepts/agent-loop.md`、`/code/openclaw/docs/concepts/queue.md`、`/code/openclaw/docs/concepts/retry.md`、`/code/openclaw/docs/gateway/sandboxing.md`、`/code/hermes-agent/run_agent.py`、`/code/hermes-agent/cron/scheduler.py`、Codex manual、Claude Code hooks/scheduled tasks 文档。

## 容易误解的点

- `agent.wait` timeout 不是 agent runtime timeout；OpenClaw 文档明确把 wait-only timeout 与 runtime abort 分开。锚点：`/code/openclaw/docs/concepts/agent-loop.md`。
- sandbox 不是“完美安全边界”；OpenClaw 文档明确说它主要降低 filesystem/process access blast radius。锚点：`/code/openclaw/docs/gateway/sandboxing.md`。
- prompt injection scanning 不能只扫创建时的字段；Hermes cron 的修复点正是扫描 user prompt 加 runtime skill content 之后的 assembled prompt。锚点：`/code/hermes-agent/cron/scheduler.py::_scan_assembled_cron_prompt`。
- 重试不是重跑整个任务；OpenClaw retry 文档明确按 HTTP request retry，避免重复非幂等操作。锚点：`/code/openclaw/docs/concepts/retry.md`。
- safe writer 不是文件写入原子性机制；Hermes 的 `_SafeWriter` 解决的是 stdout/stderr broken pipe 或 closed file。锚点：`/code/hermes-agent/run_agent.py::_SafeWriter`。
- Codex 的 read-only/workspace-write/full-access sandbox 不等于业务授权；外部 MCP tool、CI trigger、secrets 暴露和 hook trust 仍要单独建模。
- Claude Code scheduled task 的过期或禁用不是模型失败；应记录为 scheduler/policy 结果，而不是 provider error。

## 待核实问题

- OpenClaw 是否在源码层还有 Hermes `_repair_tool_call_arguments` 等价的 tool call repair 路径，本章尚未核实。
- Hermes 除 cron inactivity timeout 外，gateway/CLI 是否还有单独的 wait timeout 或 session stuck classification，本章尚未核实。
- Hermes terminal backend 的 sandbox/security 边界需要另读 `environments/**`、terminal backend 相关实现，本章只引用 README 的能力声明。
- OpenClaw prompt injection 相关实现是否有专门 scanner 或 plugin hook，本章只核实了安全文档中的 threat model 与 guardrail 定位。
