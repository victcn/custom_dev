# 自动化、Cron 与 Heartbeat

> 参考版本：
> - OpenClaw：`/code/openclaw@ca31a705d02e42ffcfb2c5884bb55339a6d0cbdc`
> - Hermes Agent：`/code/hermes-agent@3d4297a59a8607ed24850524d229f5f42520d087`
> - Codex：OpenAI Codex manual，核验于 2026-06-13：`https://developers.openai.com/codex/codex-manual.md`
> - Claude Code：Anthropic scheduled tasks 官方文档，核验于 2026-06-13：`https://code.claude.com/docs/en/scheduled-tasks`

## 本章问题

长生命周期 agent 不能只靠用户发消息触发。它还需要在未来某个时间醒来、检查外部状态、执行后台任务、把结果投递回聊天或 webhook，并且不能把后台任务和主会话上下文混在一起。这里讨论三类机制：定时调度、周期性心跳、以及从对话中推断出的后续承诺。OpenClaw 的证据来自 `docs/automation/cron-jobs.md`、`docs/gateway/heartbeat.md`、`src/cron/isolated-agent/run.ts`、`src/commitments/store.ts` 和 `src/commitments/extraction.ts`。Hermes 的证据来自 `cron/scheduler.py` 和 `cron/jobs.py`。

## OpenClaw 怎么做

OpenClaw 的 cron 是 Gateway 内置调度器；文档声明 job 定义持久化到 `~/.openclaw/cron/jobs.json`，运行状态持久化到相邻的 `jobs-state.json`，每次 cron 执行都会创建 background task 记录。锚点：`/code/openclaw/docs/automation/cron-jobs.md` 的 `How cron works`。

OpenClaw 把执行样式分为 `main`、`isolated`、`current` 和 `session:<custom-id>`。文档声明 `main` job 会向目标主会话排入 system event 并可唤醒 heartbeat；`isolated` job 会在专用 `cron:<jobId>` 会话里运行，且 isolated 的“fresh session”是每次 run 都有新的 transcript/session id，不继承旧 cron 行的 ambient conversation context。锚点：`/code/openclaw/docs/automation/cron-jobs.md` 的 `Execution styles` 与 `Main session vs isolated vs custom`。

OpenClaw isolated agent run 的源码入口是 `runCronIsolatedAgentTurn()`。它先通过 `prepareCronRunContext()` 解析 agent、session、model、delivery、tools policy 等运行上下文，再通过懒加载的 `executeCronRun()` 执行 agent，最后在 `finalizeCronRun()` 中处理输出和投递。锚点：`/code/openclaw/src/cron/isolated-agent/run.ts` 的 `prepareCronRunContext()`、`runCronIsolatedAgentTurn()`、`finalizeCronRun()`。

OpenClaw 的 isolated cron 会把投递策略显式化。源码中 `resolveCronDeliveryContext()` 解析 `announce`、`webhook`、`none`，`resolveCronToolPolicy()` 根据投递模式决定是否启用 message tool，`dispatchCronDelivery` 在收尾阶段执行 fallback delivery。锚点：`/code/openclaw/src/cron/isolated-agent/run.ts` 的 `resolveCronDeliveryContext()`、`resolveCronToolPolicy()`、`dispatchCronDelivery` 调用点。

OpenClaw 的 heartbeat 不是 detached background task，而是主会话的周期性 agent turn。文档声明 heartbeat 默认间隔是 `30m`，或在 Anthropic OAuth/token auth 模式下为 `1h`；prompt 会作为 user message 原样发送；当回复为 `HEARTBEAT_OK` 且剩余内容不超过 `ackMaxChars` 时会被当作无可见更新处理。锚点：`/code/openclaw/docs/gateway/heartbeat.md` 的 `Defaults` 与 `Response contract`。

OpenClaw 的 commitments 是源码级后续承诺机制。`buildCommitmentExtractionPrompt()` 明确要求只创建 inferred follow-up commitments，并跳过显式提醒或日程请求，因为这些属于 cron/reminders；`listDueCommitmentsForSession()` 只在 `commitments.enabled` 生效时列出到期项，并受每日上限控制。锚点：`/code/openclaw/src/commitments/extraction.ts` 的 `buildCommitmentExtractionPrompt()`；`/code/openclaw/src/commitments/store.ts` 的 `listDueCommitmentsForSession()`；`/code/openclaw/src/config/zod-schema.ts` 的 `commitments: CommitmentsSchema`。

## Hermes 怎么做

Hermes 的 cron job 存在 `~/.hermes/cron/jobs.json`，输出写到 `~/.hermes/cron/output/{job_id}/{timestamp}.md`。源码中 `JOBS_FILE`、`OUTPUT_DIR`、`parse_schedule()`、`get_due_jobs()`、`mark_job_run()`、`save_job_output()` 和 `advance_next_run()` 组成文件型 `JobStore`。锚点：`/code/hermes-agent/cron/jobs.py`。

Hermes 的调度入口是 `tick()`。它使用 `~/.hermes/cron/.tick.lock` 的文件锁避免多进程 tick 重叠，先调用 `get_due_jobs()`，再对 recurring jobs 先 `advance_next_run()`，然后按 `HERMES_CRON_MAX_PARALLEL` 或 `cron.max_parallel_jobs` 并行运行非 `workdir` jobs。锚点：`/code/hermes-agent/cron/scheduler.py` 的 `tick()`。

Hermes 的 job run 入口是 `run_job()`。`no_agent` job 会直接运行脚本并跳过 `AIAgent`；默认 LLM 路径在需要时才导入 `AIAgent`，并用 `_build_job_prompt()` 组装 prompt。锚点：`/code/hermes-agent/cron/scheduler.py` 的 `run_job()`。

Hermes 的 prompt assembly 会把 pre-run script 输出、`context_from` 上游 cron 输出、固定 cron delivery 提示和 skills 内容合并。`_build_job_prompt()` 会用 `skill_view()` 加载每个 skill，并通过 `bump_use()` 更新 skill 使用情况；组装完成后调用 `_scan_assembled_cron_prompt()` 做注入扫描。锚点：`/code/hermes-agent/cron/scheduler.py` 的 `_build_job_prompt()`、`_scan_assembled_cron_prompt()`；`/code/hermes-agent/tools/skills_tool.py` 的 `skill_view`。

Hermes 把 cron injection scanning 放在最终 assembled prompt 上，而不只扫描用户创建 job 时的 prompt 字段。`CronPromptInjectionBlocked` 的注释说明该扫描覆盖运行时加载的 skill 内容，避免恶意 skill 进入非交互 cron agent。锚点：`/code/hermes-agent/cron/scheduler.py` 的 `CronPromptInjectionBlocked` 与 `_scan_assembled_cron_prompt()`。

Hermes 的投递在 scheduler 内完成。`_resolve_delivery_targets()` 和 `_deliver_result()` 根据 job 的 delivery 配置、origin 或平台 home channel 解析目标；`tick()` 在 `run_job()` 后保存 output，再决定是否跳过 `[SILENT]` 或投递 final response。锚点：`/code/hermes-agent/cron/scheduler.py` 的 `_resolve_delivery_targets()`、`_deliver_result()`、`tick()`。

Hermes 的短生命周期 worker 体现在 cron default LLM 路径会为本次 `run_job()` 构造 `AIAgent`，并在 job 完成后由 tick 收尾保存输出和标记状态；该结论只说明 cron run 的生命周期边界，不等同于所有 Hermes agent 都是短生命周期。锚点：`/code/hermes-agent/cron/scheduler.py` 的 `run_job()`、`tick()`。

## Codex / Claude Code 参考

Codex app automations 是产品级后台任务：可以创建 standalone automations，也可以创建附着当前 thread 的 thread automations。project-scoped automations 要求本地 Codex app 正在运行、项目仍在磁盘上；Git 仓库中 automation 可选择在本地项目或 dedicated worktree 中运行，sandbox 默认沿用用户设置，后台 automation 在组织策略允许时使用 `approval_policy = "never"`（官方文档：`https://developers.openai.com/codex/codex-manual.md` 的 `Automations`）。

Codex 还把 GitHub Action、non-interactive `codex exec` 和 SDK 作为可编程自动化入口。GitHub Action 文档强调要限制触发者、控制 sandbox、保护 API key、清理 prompt injection 输入；这说明 Codex 的自动化可以在本地 app、CI runner、SDK 宿主等多种宿主里发生，而不只是一套内置 cron daemon（官方文档：`https://developers.openai.com/codex/codex-manual.md` 的 `Codex GitHub Action`、`Non-interactive mode`、`Codex SDK`）。

Claude Code 的 scheduled tasks 页说明 `/loop` 和 cron tools 可在 Claude Code session 内重复运行 prompt、轮询状态或设置一次性提醒。底层工具是 `CronCreate`、`CronList`、`CronDelete`；scheduler 每秒检查 due tasks、低优先级入队、只在 Claude 当前 turn 结束后触发；每个 session 最多 50 个 scheduled tasks，recurring tasks 默认 7 天后过期（官方文档：`https://code.claude.com/docs/en/scheduled-tasks`）。

因此外部产品参考给本章补了两个设计点：第一，自动化既可以是 detached job，也可以是 thread/session heartbeat；第二，后台任务必须绑定宿主安全策略，例如 worktree 隔离、sandbox、approval policy、触发者 allowlist、prompt injection 扫描和过期机制。

## 异同点表

| 维度 | OpenClaw | Hermes | 学习结论 |
| --- | --- | --- | --- |
| 调度归属 | Gateway 内置 scheduler；文档声明 cron 在 Gateway process 内运行。锚点：`/code/openclaw/docs/automation/cron-jobs.md` | `tick()` 是 scheduler 的一次调度入口，并用文件锁防重入；本章未展开 gateway 如何周期调用它。锚点：`/code/hermes-agent/cron/scheduler.py` | 调度器要有可重入保护；触发它的 daemon/thread 位置可以作为独立问题核实。 |
| 状态存储 | `jobs.json` + `jobs-state.json`。锚点：`/code/openclaw/docs/automation/cron-jobs.md` | `~/.hermes/cron/jobs.json` + output 目录。锚点：`/code/hermes-agent/cron/jobs.py` | 第一版 scheduler 可以先用文件存储，但要分清 job 定义、运行状态和输出归档。 |
| 隔离运行 | `isolated` job 使用 `cron:<jobId>` 并每次 fresh transcript/session。锚点：`/code/openclaw/docs/automation/cron-jobs.md`、`/code/openclaw/src/cron/isolated-agent/run.ts` | cron job 在 `run_job()` 内构造本次 `AIAgent`；没有从已读源码看到 OpenClaw 式 `cron:<jobId>` transcript 命名声明。锚点：`/code/hermes-agent/cron/scheduler.py` | 后台任务应避免污染主会话；具体隔离单位可以是专用 session、临时 agent 或 worker。 |
| 心跳 | heartbeat 是主会话周期 turn，不创建 background task。锚点：`/code/openclaw/docs/gateway/heartbeat.md` | 本章已读文件未看到等价 heartbeat 专门机制；Hermes cron 可实现周期检查。锚点：`/code/hermes-agent/cron/scheduler.py` | heartbeat 和 cron 都是自动化，但 heartbeat 更偏主会话维持，cron 更偏用户定义任务。 |
| prompt 安全 | 本章证据显示 isolated run 会处理 delivery、tool policy、model preflight；未展开 OpenClaw cron prompt scanner。锚点：`/code/openclaw/src/cron/isolated-agent/run.ts` | 明确扫描 fully assembled cron prompt，包括 runtime-loaded skills。锚点：`/code/hermes-agent/cron/scheduler.py` | 自动化任务要扫描最终组装后的 prompt，而不只是扫描用户原始字段。 |
| commitments | 源码有 inferred follow-up commitments，显式提醒交给 cron/reminders。锚点：`/code/openclaw/src/commitments/extraction.ts` | 本章已读 Hermes cron 文件未发现对应 commitments store。锚点：`/code/hermes-agent/cron/jobs.py`、`/code/hermes-agent/cron/scheduler.py` | “提醒/定时任务”和“从对话推断出的后续承诺”最好分成两个模型。 |

## Codex / Claude Code 参照表

| 维度 | Codex | Claude Code | 学习结论 |
| --- | --- | --- | --- |
| 自动化形态 | standalone/project/thread automations，可在 local project 或 worktree 运行。 | `/loop` 和 cron tools 在 session 内调度 prompt。 | 自动化既可以 detached，也可以附着当前 thread/session。 |
| 宿主要求 | project automation 依赖本地 Codex app 运行和项目磁盘可用。 | scheduler 每秒检查 due tasks，当前 turn 结束后低优先级触发。 | 定时任务的 daemon/host 生命周期必须写清楚。 |
| 权限策略 | 使用默认 sandbox；后台 automation 在策略允许时可 `approval_policy = "never"`。 | 可用环境变量关闭 cron，recurring task 有过期机制。 | unattended run 要绑定 sandbox、approval、disable switch 和 expiry。 |
| CI/外部触发 | GitHub Action、`codex exec`、SDK 可把 Codex 放进 CI/服务。 | scheduled tasks 更偏 Claude Code session 内部调度。 | 自动化入口不一定是内置 cron；CI、SDK 服务和 wrapper 都是入口。 |

## 源码阅读路线

1. OpenClaw 先读 `/code/openclaw/docs/automation/cron-jobs.md`，确认 schedule types、execution styles、delivery modes 和 config 字段。
2. OpenClaw 再读 `/code/openclaw/docs/gateway/heartbeat.md`，确认 heartbeat 与 cron 的边界、`HEARTBEAT_OK` contract、`isolatedSession`。
3. OpenClaw 接着读 `/code/openclaw/src/cron/isolated-agent/run.ts` 的 `prepareCronRunContext()`、`runCronIsolatedAgentTurn()`、`finalizeCronRun()`。
4. OpenClaw commitments 读 `/code/openclaw/src/commitments/extraction.ts` 的 `buildCommitmentExtractionPrompt()`，再读 `/code/openclaw/src/commitments/store.ts` 的 `listDueCommitmentsForSession()`。
5. Hermes 先读 `/code/hermes-agent/cron/jobs.py` 的 `parse_schedule()`、`add_job()`、`get_due_jobs()`、`advance_next_run()`。
6. Hermes 再读 `/code/hermes-agent/cron/scheduler.py` 的 `tick()`、`run_job()`、`_build_job_prompt()`、`_scan_assembled_cron_prompt()`、`_deliver_result()`。
7. Codex 读 manual 的 `Automations`、`Codex GitHub Action`、`Non-interactive mode`、`Codex SDK`，确认 app automation、CI automation、SDK automation 的不同宿主。
8. Claude Code 读 `https://code.claude.com/docs/en/scheduled-tasks`，确认 `/loop`、`CronCreate`/`CronList`/`CronDelete`、低优先级入队、session task 上限和过期策略。

## 最小复现抽象

这是学习抽象，不是 OpenClaw、Hermes、Codex 或 Claude Code 的精确实现。

- `Scheduler`：周期 tick，找 due jobs，控制并发和 at-most-once；对应 OpenClaw Gateway cron 文档与 Hermes `tick()`。
- `JobStore`：持久化 job 定义、运行状态和输出；对应 OpenClaw `jobs.json`/`jobs-state.json` 文档与 Hermes `cron/jobs.py`。
- `PromptAssembler`：合并 job prompt、pre-run script 输出、上游 job output、skills 和 cron 运行规则；对应 Hermes `_build_job_prompt()`，OpenClaw isolated run 的 prompt 细节需继续深入。
- `RunLauncher`：根据 `main`/`isolated`/custom session 决定是排主会话事件还是启动独立 run；对应 OpenClaw `runCronIsolatedAgentTurn()` 与 Hermes `run_job()`。
- `DeliveryDispatcher`：根据 `announce`、`webhook`、`none` 或平台 home channel 做结果投递，并处理 silent marker；对应 OpenClaw `dispatchCronDelivery` 调用点与 Hermes `_deliver_result()`。
- `AutomationHostPolicy`：绑定 worktree/local project、sandbox、approval policy、触发者 allowlist、过期时间和 prompt injection 扫描；Codex automations/GitHub Action 与 Claude scheduled tasks 都证明该策略必须在 agent run 之前确定。

## 容易误解的点

- heartbeat 不是 cron 的同义词。OpenClaw 文档明确 heartbeat 是主会话周期 turn，cron isolated job 是 detached work。锚点：`/code/openclaw/docs/gateway/heartbeat.md`、`/code/openclaw/docs/automation/cron-jobs.md`。
- OpenClaw commitments 不是显式 reminder。源码 prompt 明确跳过“remind me tomorrow”等显式请求，因为那类请求属于 cron/reminders。锚点：`/code/openclaw/src/commitments/extraction.ts` 的 `buildCommitmentExtractionPrompt()`。
- Hermes cron 的 skill loading 不是只把 skill 名字写进 prompt；`_build_job_prompt()` 读取完整 skill 内容并在 assembled prompt 上扫描。锚点：`/code/hermes-agent/cron/scheduler.py`。
- `[SILENT]` 在 Hermes 是投递抑制标记，不代表 job 没有被记录；`tick()` 仍会保存 output 并 `mark_job_run()`。锚点：`/code/hermes-agent/cron/scheduler.py`。
- Codex automation 不是一定在云端运行；官方文档说明 project-scoped automation 依赖本地 Codex app、磁盘项目和可选 worktree。
- Claude Code scheduled task 不是随时打断当前回复；官方文档说明 scheduled prompt 在 Claude 忙时等待当前 turn 结束，并以低优先级入队。

## 待核实问题

- OpenClaw cron prompt 的构造是否有独立 prompt-injection scanner，需要继续读 `src/cron/isolated-agent/*` 之外的 executor/context runtime。
- OpenClaw commitments 是否有面向用户的独立 docs 页面；本章只核实到源码与 config schema。
- Hermes gateway 如何周期调用 `cron.scheduler.tick()` 的具体线程位置，本章未展开 `gateway` 入口。
