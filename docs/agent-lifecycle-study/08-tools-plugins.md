# 工具、插件与 Provider

> 参考版本：
> - OpenClaw：`/code/openclaw@ca31a705d02e42ffcfb2c5884bb55339a6d0cbdc`
> - Hermes Agent：`/code/hermes-agent@3d4297a59a8607ed24850524d229f5f42520d087`
> - Codex / Claude Code：作为 MCP、SDK/API tool use、plugins、hooks、skills/subagents 与 coding-agent 扩展面的官方文档参照，核验于 2026-06-13：`https://developers.openai.com/codex/codex-manual.md`；`https://platform.claude.com/docs/en/agents-and-tools/mcp-connector`；`https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview`

## 本章问题

长生命周期 agent 必须把“能做什么”和“谁允许它做”分层：工具是执行动作的函数，插件扩展工具、provider、hooks 或渠道，MCP 把外部工具/数据源以协议形式接进 agent host，安全扫描和审批控制风险。本章比较已读证据：OpenClaw 的 `docs/tools/index.md`、`docs/plugins/hooks.md`、`src/hooks/plugin-hooks.ts`，Hermes 的 `tools/registry.py`、`providers/README.md`、`providers/__init__.py`、`providers/base.py`、`tools/skills_guard.py` 和 `tools/skill_manager_tool.py`；Codex/Claude Code 仅用官方文档参照 MCP 与 API/SDK tool surface。

## OpenClaw 怎么做

OpenClaw 文档把 tools、skills、plugins 分成三层：tool 是模型可调用的 typed function，skill 是注入 system prompt 的 `SKILL.md` 指导，plugin 可以注册 channels、model providers、tools、skills、speech、media、web fetch/search 等能力。锚点：`/code/openclaw/docs/tools/index.md` 的 `Tools, skills, and plugins`。

OpenClaw 的工具面包括内置工具和插件工具。文档列出 `exec`、`browser`、`web_search`、`read`、`write`、`edit`、`apply_patch`、`message`、`cron`、`gateway`、`sessions_*`、`subagents` 等内置工具；插件工具通过 `api.registerTool(...)` 注册，并在 manifest 的 `contracts.tools` 中声明。锚点：`/code/openclaw/docs/tools/index.md` 的 `Built-in tools` 与 `Plugin-provided tools`。

OpenClaw 的工具授权由 `tools.allow`、`tools.deny`、`tools.profile`、`tools.byProvider` 等配置控制；文档声明 deny 优先于 allow，显式 allowlist 如果解析不到任何可调用工具会 fail closed。锚点：`/code/openclaw/docs/tools/index.md` 的 `Tool configuration` 与 `Provider-specific restrictions`。

OpenClaw 的 plugin hooks 是 in-process extension points。文档列出 agent turn、tools、messages and delivery、sessions、subagents、lifecycle 等 hook catalog，其中 `before_tool_call` 可改写参数、block 或 require approval，`after_tool_call` 观察结果，`before_dispatch` 和 `reply_dispatch` 参与 outbound dispatch pipeline。锚点：`/code/openclaw/docs/plugins/hooks.md` 的 `Hook catalog`、`Tool call policy`。

OpenClaw 的 hook 发现会检查插件是否激活、memory slot 是否允许、hook path 是否存在并且真实路径没有逃出 plugin root。源码 `resolvePluginHookDirs()` 使用 `loadPluginMetadataSnapshot()`、`resolveEffectivePluginActivationState()`、`resolveMemorySlotDecision()` 和 `isPathInsideWithRealpath()`。锚点：`/code/openclaw/src/hooks/plugin-hooks.ts` 的 `resolvePluginHookDirs()`。

OpenClaw provider plugin 的证据来自插件 SDK 文档。`docs/tools/index.md` 声明插件可注册 model providers；`docs/plugins/sdk-entrypoints.md` 示例使用 `api.registerProvider(...)`；`docs/plugins/sdk-subpaths.md` 列出 provider runtime/auth/catalog/stream/tool-compat 等 SDK subpaths。锚点：`/code/openclaw/docs/tools/index.md`、`/code/openclaw/docs/plugins/sdk-entrypoints.md`、`/code/openclaw/docs/plugins/sdk-subpaths.md`。

## Hermes 怎么做

Hermes 的 `ToolRegistry` 是工具集中注册表。每个工具文件在模块顶层调用 `registry.register()` 声明 schema、handler、toolset、availability check 等，`discover_builtin_tools()` 用 AST 找顶层 `registry.register(...)` 并导入对应模块。锚点：`/code/hermes-agent/tools/registry.py` 的文件注释、`discover_builtin_tools()`、`ToolRegistry.register()`。

Hermes 的 tool registry 会防止非 MCP 工具互相 shadow。`register()` 发现同名工具来自不同 toolset 时，只有双方都是 `mcp-*` 才允许覆盖，否则记录 error 并拒绝注册。锚点：`/code/hermes-agent/tools/registry.py` 的 `ToolRegistry.register()`。

Hermes 的工具定义生成会执行 availability check，并把 `check_fn` 结果 TTL 缓存约 30 秒；`get_definitions()` 只返回通过 `check_fn` 的 OpenAI-format tool schemas，并支持 `dynamic_schema_overrides`。锚点：`/code/hermes-agent/tools/registry.py` 的 `_CHECK_FN_TTL_SECONDS`、`_check_fn_cached()`、`ToolRegistry.get_definitions()`。

Hermes 的工具执行由 `dispatch()` 统一调用 handler。未知工具返回 JSON error，handler 异常会被捕获并序列化成 `{"error": ...}`。锚点：`/code/hermes-agent/tools/registry.py` 的 `ToolRegistry.dispatch()`、`tool_error()`、`tool_result()`。

Hermes 的 provider registry 使用 `ProviderProfile`。`providers/README.md` 声明 provider profile 是 auth、transport kwargs、model listing、runtime routing 的单一数据源；`providers/__init__.py` 懒加载 bundled plugin、user plugin 和 legacy single-file provider，用户插件可覆盖 bundled provider。锚点：`/code/hermes-agent/providers/README.md`、`/code/hermes-agent/providers/__init__.py`、`/code/hermes-agent/providers/base.py`。

Hermes 的 skill safety scanning 在 `tools/skills_guard.py`。该 scanner 对外部 skill 做 regex 静态分析，覆盖 exfiltration、prompt injection、destructive、persistence、network、obfuscation 等类别，并按 `builtin`、`trusted`、`community`、`agent-created` 信任级别决定 allow/block/ask。锚点：`/code/hermes-agent/tools/skills_guard.py` 的模块注释、`THREAT_PATTERNS`、`INSTALL_POLICY`。

Hermes agent-created skill 的扫描默认关闭；`tools/skill_manager_tool.py` 的 `_guard_agent_created_enabled()` 读取 `skills.guard_agent_created`，开启后 `_security_scan_skill()` 会调用 `scan_skill()` 和 `should_allow_install()`，被阻断时回滚写入。锚点：`/code/hermes-agent/tools/skill_manager_tool.py` 的 `_guard_agent_created_enabled()`、`_security_scan_skill()`、`_create_skill()` 和 `_edit_skill()` 附近的 security scan。

## Codex / Claude Code 参照

Codex 和 Claude Code 把 MCP 作为重要扩展面。Codex MCP 文档说明 `mcp_servers` 可配置外部 MCP servers，并支持 `stdio` 与 `streamable_http` transport；Codex CLI reference 还说明 `codex mcp-server` 可把 Codex 作为 MCP server over stdio 暴露给其他工具：`https://developers.openai.com/codex/mcp`；`https://developers.openai.com/codex/cli/reference`。

Claude 侧要区分 MCP 与 Claude API tool use。Anthropic MCP connector 文档说明 Messages API 可以连接 remote MCP servers；tool use 文档则描述 Claude 在 HTTP Messages API 中产生 structured tool requests，由应用或 server-side tool 执行。前者是外部工具/数据源协议，后者是模型 API 的工具调用契约：`https://platform.claude.com/docs/en/agents-and-tools/mcp-connector`；`https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview`。

这给本章的学习抽象补了一条边界：OpenClaw 的 plugin hooks、Hermes 的 registry/profile、Codex/Claude Code 的 MCP/API tool use 都是在回答同一个问题，即“agent host 怎样发现工具、授权工具、调用工具、审计工具”。区别在于：有的系统把扩展做成 in-process plugin，有的做成本地 registry，有的做成协议 server。

## 异同点表

| 维度 | OpenClaw | Hermes | 学习结论 |
| --- | --- | --- | --- |
| 工具模型 | 文档强调工具是模型 API 里的 typed function，插件可注册工具。锚点：`/code/openclaw/docs/tools/index.md` | 工具模块顶层 `registry.register()`，注册表统一给 `model_tools.py` 查询。锚点：`/code/hermes-agent/tools/registry.py` | 工具应集中注册、集中导出 schema，避免 agent loop 直接依赖每个工具实现。 |
| 插件扩展 | plugin 可注册 channels、providers、tools、skills、media 等。锚点：`/code/openclaw/docs/tools/index.md` | provider profile 以 bundled/user plugin 目录加载；tool registry 本身不是 OpenClaw 式通用 plugin hook bus。锚点：`/code/hermes-agent/providers/__init__.py` | 插件可以是统一扩展系统，也可以按 provider/tool/skill 分域实现；关键是边界明确。 |
| HookBus | `api.on(...)` 注册 hook，覆盖 agent/tool/message/session/subagent/lifecycle。锚点：`/code/openclaw/docs/plugins/hooks.md` | 本章已读 Hermes 证据没有通用 hook bus；工具和 provider 通过 registry/profile 扩展。锚点：`/code/hermes-agent/tools/registry.py`、`/code/hermes-agent/providers/__init__.py` | HookBus 是高级能力，不是最小 agent loop 的前置条件。 |
| Tool lifecycle hooks | `before_tool_call`、`after_tool_call`、`tool_result_persist` 等。锚点：`/code/openclaw/docs/plugins/hooks.md` | registry 的生命周期是 register、get_definitions、dispatch；没有已核实的 before/after hook catalog。锚点：`/code/hermes-agent/tools/registry.py` | 工具生命周期 hook 适合做审计、审批和结果净化；没有 hook 时也要保留 registry 层边界。 |
| Provider | provider plugin 可通过 SDK 注册。锚点：`/code/openclaw/docs/plugins/sdk-entrypoints.md` | `ProviderProfile` 懒发现并注册，`run_agent.py` 传入 profile 给 transport。锚点：`/code/hermes-agent/providers/README.md` | provider registry 能把模型供应商差异从 agent loop 中移出。 |
| 安全扫描 | plugin hook 可在 `before_install` 检查 skill/plugin install scan 并 block；hook path 不能逃出 plugin root。锚点：`/code/openclaw/docs/plugins/hooks.md`、`/code/openclaw/src/hooks/plugin-hooks.ts` | `skills_guard.py` 对 skill 内容做静态扫描，`skill_manager_tool.py` 对 agent-created skill 可选启用扫描。锚点：`/code/hermes-agent/tools/skills_guard.py`、`/code/hermes-agent/tools/skill_manager_tool.py` | 长生命周期 agent 会安装和生成扩展内容，安装前扫描与运行时审批都需要。 |
| MCP/API tool use | 本章未核实 OpenClaw 是否把 MCP 作为一等扩展协议；已读证据更偏 plugin hooks 和 node caps/commands。 | 已读 Hermes 证据中有 `mcp-*` toolset shadow 特例，但本章未展开 MCP adapter。锚点：`/code/hermes-agent/tools/registry.py` | Codex/Claude Code 提醒：MCP 可作为跨 agent host 的工具协议；不要只按本项目私有 plugin 设计。 |

## Codex / Claude Code 参照表

| 维度 | Codex | Claude Code | 学习结论 |
| --- | --- | --- | --- |
| MCP client | CLI/IDE 支持 stdio 和 streamable HTTP MCP servers，配置在 `config.toml`。 | Claude API MCP connector 可连接 remote MCP servers；Claude Code 也围绕 MCP 扩展工具。 | MCP 是跨 host 工具协议，不等同于某项目私有 plugin。 |
| MCP server | `codex mcp-server` 可让其他工具连接 Codex；app-server 是本地 JSON-RPC/transport surface。 | Agent SDK 宿主可组合工具、MCP 和 API tool use。 | 一个系统既可能是 MCP client，也可能向外暴露能力。 |
| tool approval | MCP tools 有 allow/deny、timeout、approval mode 等配置。 | hooks/permissions 可在 tool use 前后拦截或审计。 | tool registry 必须和权限、超时、审计绑定。 |
| plugin 分发 | Codex plugin 可携带 skills、MCP server config、hooks、apps。 | Claude Code plugin skills 有 namespace，hooks/subagents/skills 构成扩展面。 | plugin 是分发和信任边界，skill/tool/MCP 是内部能力类型。 |

## 源码阅读路线

1. OpenClaw 先读 `/code/openclaw/docs/tools/index.md`，建立 tools、skills、plugins 三层模型。
2. OpenClaw 再读 `/code/openclaw/docs/plugins/hooks.md`，重点看 `Hook catalog`、`Tool call policy`、`Prompt and model hooks`、`before_install`。
3. OpenClaw 接着读 `/code/openclaw/src/hooks/plugin-hooks.ts` 的 `resolvePluginHookDirs()`，确认 plugin hook path 和激活策略。
4. OpenClaw provider 继续读 `/code/openclaw/docs/plugins/sdk-entrypoints.md` 和 `/code/openclaw/docs/plugins/sdk-subpaths.md`。
5. Hermes 先读 `/code/hermes-agent/tools/registry.py` 的 `discover_builtin_tools()`、`ToolRegistry.register()`、`get_definitions()`、`dispatch()`。
6. Hermes provider 读 `/code/hermes-agent/providers/README.md`，再读 `/code/hermes-agent/providers/__init__.py` 和 `/code/hermes-agent/providers/base.py`。
7. Hermes 安全扫描读 `/code/hermes-agent/tools/skills_guard.py`，再读 `/code/hermes-agent/tools/skill_manager_tool.py` 的 `_security_scan_skill()`。
8. Codex/Claude Code 参照读 `02-gateway.md` 的 MCP 小节，再回到官方 Codex CLI configuration/reference、Claude MCP connector 和 Claude tool use 文档，区分 MCP server、MCP client、API tool use 与本地 registry/plugin。

## 最小复现抽象

这是学习抽象，不是 OpenClaw、Hermes、Codex 或 Claude Code 的精确实现。

- `ToolRegistry`：接收工具 schema、handler、toolset、availability check，提供 `get_definitions()` 和 `dispatch()`；主要对应 Hermes `ToolRegistry`，OpenClaw 对应工具系统的公开文档。
- `PluginRegistry`：发现插件 manifest、激活状态和插件导出的 capabilities；对应 OpenClaw `resolvePluginHookDirs()` 与 provider SDK 文档，Hermes provider plugin discovery 可视作 provider 专用 registry。
- `MCPBridge`：把远程或本地 MCP server 暴露的 tools/resources/prompts 接进 agent host，也可以把 agent host 自身作为 MCP server 暴露给其他工具；对应 Codex/Claude Code 的官方 MCP 文档参照，Hermes 的 `mcp-*` toolset 需要后续继续核实。
- `HookBus`：按 hook name、priority、timeout 顺序执行 handler，允许 block、rewrite、approval 或 observe；对应 OpenClaw `api.on(...)` 文档。
- `ProviderRegistry`：按 provider name/alias 返回 profile，并允许用户覆盖 bundled profile；对应 Hermes `providers/__init__.py` 和 OpenClaw provider plugin SDK。
- `SafetyScanner`：安装或加载前扫描 skill/plugin/context，输出 allow/block/ask；对应 Hermes `skills_guard.py` 和 OpenClaw `before_install` hook 与 hook path realpath 检查。

## 容易误解的点

- skill 不是 tool。OpenClaw 文档明确 skill 是 `SKILL.md` prompt 指导，tool 是模型可调用函数；Hermes 的 `skill_manager_tool.py` 也把 skill 定义为 procedural memory。锚点：`/code/openclaw/docs/tools/index.md`、`/code/hermes-agent/tools/skill_manager_tool.py`。
- OpenClaw plugin hook 不是 shell hook。`docs/plugins/hooks.md` 明确 plugin hooks 是 in-process extension points，并区分 internal hooks。锚点：`/code/openclaw/docs/plugins/hooks.md`。
- Hermes provider plugin 不等于任意工具插件。已读证据显示 provider plugin 目录专门注册 `ProviderProfile`；工具扩展仍走 `ToolRegistry`。锚点：`/code/hermes-agent/providers/__init__.py`、`/code/hermes-agent/tools/registry.py`。
- MCP 不是 provider registry，也不是模型 API 本身。MCP 是 agent host 与外部工具/数据源之间的协议；Claude API tool use 是 Messages API 内的工具调用契约；两者常一起出现但边界不同。
- Hermes skill scanner 对 agent-created skill 默认不开启；不能把它描述成所有本地 skill 写入都强制扫描。锚点：`/code/hermes-agent/tools/skill_manager_tool.py` 的 `_guard_agent_created_enabled()`。

## 待核实问题

- OpenClaw plugin install 的 built-in scan 具体实现位置和扫描规则，本章只核实到 `before_install` hook 文档与 hook path realpath 检查。
- Hermes 是否存在通用 plugin hook bus，以及 MCP adapter 的完整接入路径，需要继续阅读 `hermes_cli/plugins`、gateway platform registry 和 MCP toolset 相关文件。
- OpenClaw provider plugin 的 runtime 注册源码尚未系统阅读，本章只引用 SDK 文档和工具总览。
