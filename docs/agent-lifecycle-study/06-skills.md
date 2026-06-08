# Skills 机制

> 参考版本：
> - OpenClaw：`/code/openclaw@ca31a705d02e42ffcfb2c5884bb55339a6d0cbdc`
> - Hermes Agent：`/code/hermes-agent@3d4297a59a8607ed24850524d229f5f42520d087`

## 本章问题

Skills 是长生命周期 agent 的过程记忆：它们把“怎么做某类任务”的经验从一次对话中沉淀成可再次加载的 `SKILL.md`，避免每次靠模型临场推理或用户重复纠正。[学习抽象；锚点：`/code/openclaw/docs/tools/skills.md:11`、`/code/hermes-agent/tools/skill_manager_tool.py:797`]

本章关注四个问题：skills 从哪里加载、如何进入 prompt 或工具视图、如何创建/更新、如何维护生命周期。OpenClaw 更强调多来源加载、优先级、插件技能边界和 snapshot；Hermes 更强调 `SKILLS_GUIDANCE`、`skill_manage` 自我更新、usage telemetry 和 curator 后台维护。[源码事实/文档声明；锚点：`/code/openclaw/docs/tools/skills.md:17`、`/code/hermes-agent/agent/prompt_builder.py:179`、`/code/hermes-agent/agent/curator.py:1`]

## OpenClaw 怎么做

OpenClaw 使用 AgentSkills-compatible skill folders，每个 skill 是一个包含 `SKILL.md` 的目录，`SKILL.md` 带 YAML frontmatter 和 instructions；OpenClaw 在 load time 根据环境、config 和 binary presence 过滤 skill。[文档声明；锚点：`/code/openclaw/docs/tools/skills.md:11`]

OpenClaw 文档声明的加载来源和优先级从高到低是：`<workspace>/skills`、`<workspace>/.agents/skills`、`~/.agents/skills`、`~/.openclaw/skills`、bundled skills、`skills.load.extraDirs`。[文档声明；锚点：`/code/openclaw/docs/tools/skills.md:17`]

OpenClaw 源码中的 `loadWorkspaceSkillEntries` 路径解析与文档一致：它解析 `managedSkillsDir`、`workspaceSkillsDir`、`bundledSkillsDir`、`pluginSkillsDir`、`skills.load.extraDirs`，再把 `extra < bundled < managed < agents-skills-personal < agents-skills-project < workspace` 按覆盖顺序合并。[源码事实；锚点：`/code/openclaw/src/agents/skills/workspace.ts:712`、`/code/openclaw/src/agents/skills/workspace.ts:768`]

OpenClaw 区分 skill location 和 visibility：位置/优先级决定同名 skill 哪个版本生效，agent allowlists 决定某个 agent 实际能用哪些 skills；非空 `agents.list[].skills` 是最终集合，不与 defaults 合并。[文档声明；锚点：`/code/openclaw/docs/tools/skills.md:56`、`/code/openclaw/docs/tools/skills.md:77`]

OpenClaw 的 plugins 和 skills 有明确边界：plugin 可以在 `openclaw.plugin.json` 里列出 `skills` 目录，plugin enabled 时这些 skills 才加载；plugin skill directories 被合并到与 `skills.load.extraDirs` 同样的低优先级路径。[文档声明；锚点：`/code/openclaw/docs/tools/skills.md:89`]

OpenClaw 源码通过 `resolvePluginSkillDirs()` 从 plugin metadata snapshot 解析 enabled plugin 的 skill paths，并用 realpath 检查防止 path escape；`publishPluginSkills()` 把 plugin skill 作为 OpenClaw 管理的 symlink 发布到 plugin skills directory，注释说明它们以 extra-dir precedence 发布，不能 shadow managed 或 bundled skills。[源码事实；锚点：`/code/openclaw/src/agents/skills/plugin-skills.ts:22`、`/code/openclaw/src/agents/skills/plugin-skills.ts:94`、`/code/openclaw/src/agents/skills/plugin-skills.ts:193`、`/code/openclaw/src/agents/skills/plugin-skills.ts:213`]

OpenClaw `SKILL.md` 至少需要 `name` 和 `description` frontmatter；文档说明 embedded agent parser 支持 single-line frontmatter keys，并建议 `metadata` 使用 single-line JSON object。[文档声明；锚点：`/code/openclaw/docs/tools/skills.md:166`]

OpenClaw skill frontmatter 支持 `user-invocable`、`disable-model-invocation`、`command-dispatch`、`command-tool` 和 `command-arg-mode` 等键，使 skill 可以从普通 prompt instructions 变成 slash command 或直接 tool dispatch。[文档声明；锚点：`/code/openclaw/docs/tools/skills.md:182`]

OpenClaw `skills.entries` 可以启停 bundled/managed skills，注入 `apiKey`、`env` 和 custom `config`；agent run 开始时读取 metadata、应用 env/API key、构建 eligible skills prompt，并在 run 结束后恢复环境。[文档声明；锚点：`/code/openclaw/docs/tools/skills.md:319`、`/code/openclaw/docs/tools/skills.md:378`]

OpenClaw snapshot eligible skills at session start，并在同一 session 复用；文档说明 skills watcher 默认会在 `SKILL.md` 改动时 bump snapshot，下一 turn 可热刷新。[文档声明；锚点：`/code/openclaw/docs/tools/skills.md:397`]

OpenClaw 的 Skill Workshop 是 optional experimental plugin，可从 agent work 观察到的 reusable procedures 创建或更新 workspace skills；它只写 `<workspace>/skills`，扫描生成内容，支持 pending approval 或 automatic safe writes，并刷新 skill snapshot。[文档声明；锚点：`/code/openclaw/docs/tools/skills.md:106`]

## Hermes 怎么做

Hermes 的 skills 主目录是 `~/.hermes/skills`；`tools/skills_tool.py` 注释说明 agent edits、hub installs 和 bundled skills 都 coexist here，不污染 git repo。[源码事实；锚点：`/code/hermes-agent/tools/skills_tool.py:85`]

Hermes 的 `skills_list()` 是 progressive disclosure 第一层，只返回 name、description、category 等 minimal metadata，并提示用 `skill_view(name)` 查看完整内容。[源码事实；锚点：`/code/hermes-agent/tools/skills_tool.py:674`]

Hermes 的 `skill_view()` 支持普通 skill、category path、外部目录 skill，以及 `plugin:skill` 形式的 plugin-provided skills；它也支持读取 skill 目录内 `references/`、`templates/`、`assets/`、`scripts/` 等 linked files。[源码事实；锚点：`/code/hermes-agent/tools/skills_tool.py:849`、`/code/hermes-agent/tools/skills_tool.py:872`、`/code/hermes-agent/tools/skills_tool.py:1086`]

Hermes 的 skill discovery 会先看本地 `SKILLS_DIR`，再看 `skills.external_dirs`；`get_external_skills_dirs()` 从 `config.yaml` 读取外部目录，展开 `~`/env vars，解析为绝对路径，并跳过不存在或重复路径。[源码事实；锚点：`/code/hermes-agent/tools/skills_tool.py:939`、`/code/hermes-agent/agent/skill_utils.py:187`、`/code/hermes-agent/agent/skill_utils.py:273`]

Hermes 的 `SKILLS_GUIDANCE` 会在 `skill_manage` 可用时注入 system prompt，要求复杂任务、tricky error 或非平凡 workflow 后把方法保存为 skill；如果使用 skill 时发现过时、缺失或错误，要立即 `skill_manage(action='patch')`。[源码事实；锚点：`/code/hermes-agent/agent/prompt_builder.py:179`、`/code/hermes-agent/run_agent.py:5647`]

Hermes 的 skills prompt 会把可用 skills 放入 `<available_skills>`，并要求模型在任务相关时先 `skill_view(name)`；如果 skill 有问题，要用 `skill_manage(action='patch')` 修复，困难/迭代任务后要考虑保存为 skill。[源码事实；锚点：`/code/hermes-agent/agent/prompt_builder.py:1175`]

Hermes 的 `skill_manage()` 统一处理 `create`、`edit`、`patch`、`delete`、`write_file`、`remove_file`；其 schema 描述把 skills 定义为 procedural memory，并声明新 skills 写到 `~/.hermes/skills/`。[源码事实；锚点：`/code/hermes-agent/tools/skill_manager_tool.py:713`、`/code/hermes-agent/tools/skill_manager_tool.py:797`]

Hermes 创建 skill 时会验证名称、category、frontmatter 和内容大小，检查同名冲突，创建目录并 atomically 写 `SKILL.md`；security scan 失败时会回滚目录。[源码事实；锚点：`/code/hermes-agent/tools/skill_manager_tool.py:373`]

Hermes 更新 skill 时优先支持 `patch`：`_patch_skill()` 在 `SKILL.md` 或 supporting file 内做 targeted find-and-replace，使用与 file patch tool 相同的 fuzzy matching，并在 patch `SKILL.md` 后验证 frontmatter。[源码事实；锚点：`/code/hermes-agent/tools/skill_manager_tool.py:463`]

Hermes 的 `skills_hub.py` 是 Skills Hub library module，提供 `SkillSource` adapter、官方 optional skills、GitHub source、lock file、quarantine、audit log、taps 和 index cache；安装 provenance 由 `.hub/lock.json` 管理。[源码事实；锚点：`/code/hermes-agent/tools/skills_hub.py:1`、`/code/hermes-agent/tools/skills_hub.py:48`、`/code/hermes-agent/tools/skills_hub.py:294`]

Hermes 的 usage telemetry 写在 `~/.hermes/skills/.usage.json`，记录 use/view/patch counters 和 lifecycle state；设计注释明确它是 sidecar，不写进 `SKILL.md`，并且 bundled/hub-installed skills 不进入 curator 管理范围。[源码事实；锚点：`/code/hermes-agent/tools/skill_usage.py:1`、`/code/hermes-agent/tools/skill_usage.py:8`、`/code/hermes-agent/tools/skill_usage.py:151`]

Hermes 的 curator 是 inactivity-triggered background skill maintenance orchestrator，不是 cron daemon；当 agent idle 且距离上次 curator run 超过 `interval_hours` 时，`maybe_run_curator()` 会 fork 一个 `AIAgent` 评审 skills。[源码事实；锚点：`/code/hermes-agent/agent/curator.py:1`]

Hermes curator 的职责包括按 skill activity timestamps 自动迁移 lifecycle state、通过 `skill_manage` pin/archive/consolidate/patch agent-created skills、把状态保存在 `.curator_state`；strict invariants 包括只触碰 agent-created skills、不自动删除只 archive、pinned skills 跳过 auto-transitions、使用 auxiliary client。[源码事实；锚点：`/code/hermes-agent/agent/curator.py:9`、`/code/hermes-agent/agent/curator.py:15`]

Hermes 的生命周期状态是 `active`、`stale`、`archived`，另有 `pinned` 布尔标记；`apply_automatic_transitions()` 根据 `stale_after_days` 和 `archive_after_days` 把 agent-created skills 标为 stale、archive 或 reactivated。[源码事实；锚点：`/code/hermes-agent/tools/skill_usage.py:18`、`/code/hermes-agent/agent/curator.py:256`]

## 异同点表

| 维度 | OpenClaw | Hermes | 学习结论 |
| --- | --- | --- | --- |
| skill 格式 | AgentSkills-compatible folder + `SKILL.md`，frontmatter 至少 `name`/`description`。[锚点：`/code/openclaw/docs/tools/skills.md:11`、`/code/openclaw/docs/tools/skills.md:166`] | `skills_tool.py` 使用 `SKILL.md` + frontmatter + references/templates/assets/scripts 的 progressive disclosure 结构。[锚点：`/code/hermes-agent/tools/skills_tool.py:1`] | `SKILL.md` 是两者共享的过程记忆载体，目录结构可逐步扩展。 |
| 加载根 | workspace/project-agent/personal/managed/bundled/extraDirs，多级优先级。[锚点：`/code/openclaw/docs/tools/skills.md:17`] | 本地 `~/.hermes/skills` first，然后 `skills.external_dirs`；plugin skills 可用 `plugin:skill` 路由。[锚点：`/code/hermes-agent/tools/skills_tool.py:85`、`/code/hermes-agent/agent/skill_utils.py:273`] | 多来源加载必须有明确优先级，否则同名 skill 会产生不可预测覆盖。 |
| 插件边界 | Plugin skills 来自 plugin manifest 的 `skills` 目录，enabled plugin 才加载，并以 extra-dir 低优先级发布。[锚点：`/code/openclaw/docs/tools/skills.md:89`、`/code/openclaw/src/agents/skills/plugin-skills.ts:213`] | `skill_view()` 对 `plugin:skill` 做 qualified name dispatch，走 plugin manager 查找。[锚点：`/code/hermes-agent/tools/skills_tool.py:872`] | skill 可以由插件提供，但运行时仍要经过统一 discovery/visibility 规则。 |
| Prompt 策略 | Eligible skills snapshot 进入 prompt；`disable-model-invocation` 可阻止普通 prompt 注入。[锚点：`/code/openclaw/docs/tools/skills.md:190`、`/code/openclaw/docs/tools/skills.md:397`] | `build_skills_system_prompt()` 生成 mandatory skills block，要求相关时先 `skill_view()`。[锚点：`/code/hermes-agent/agent/prompt_builder.py:1175`] | 不应把所有 skill 全量注入；更稳妥的是先索引，再按需查看全文。 |
| 创建/更新 | Skill Workshop plugin 可创建/更新 workspace skills，默认 disabled。[锚点：`/code/openclaw/docs/tools/skills.md:106`] | `skill_manage()` 直接提供 create/edit/patch/delete/write_file/remove_file。[锚点：`/code/hermes-agent/tools/skill_manager_tool.py:713`] | 让 agent 写 skill 前应有明确权限边界和审计路径。 |
| 生命周期维护 | 本章核实到 snapshot/watch/hot refresh 和 ClawHub install/update；未核实到类似 Hermes curator 的自动 stale/archive 生命周期。[锚点：`/code/openclaw/docs/tools/skills.md:397`、`/code/openclaw/docs/tools/skills.md:123`] | curator 根据 usage sidecar 和 inactivity gate 做 stale/archive/consolidation。[锚点：`/code/hermes-agent/agent/curator.py:1`、`/code/hermes-agent/tools/skill_usage.py:18`] | 自我改进 skill 的难点不在创建，而在长期维护、归档和去重。 |

## 源码阅读路线

1. OpenClaw 文档入口：`/code/openclaw/docs/tools/skills.md`，按 Locations、Agent allowlists、Plugins and skills、Skill Workshop、SKILL.md format、Snapshots 阅读。[锚点：`/code/openclaw/docs/tools/skills.md:17`、`/code/openclaw/docs/tools/skills.md:89`、`/code/openclaw/docs/tools/skills.md:397`]
2. OpenClaw plugin 边界：`/code/openclaw/docs/plugins/architecture.md` 读 capability/ownership，把 plugin 理解成 feature/company ownership boundary；skills 只是 plugin 可携带的一类指导材料。[锚点：`/code/openclaw/docs/plugins/architecture.md:31`]
3. OpenClaw loader 实现：`/code/openclaw/src/agents/skills/workspace.ts`，读 `loadWorkspaceSkillEntries()` 的 root 解析与 merge precedence。[锚点：`/code/openclaw/src/agents/skills/workspace.ts:712`]
4. OpenClaw plugin skills：`/code/openclaw/src/agents/skills/plugin-skills.ts`，读 `resolvePluginSkillDirs()` 和 `publishPluginSkills()`。[锚点：`/code/openclaw/src/agents/skills/plugin-skills.ts:22`、`/code/openclaw/src/agents/skills/plugin-skills.ts:201`]
5. Hermes prompt guidance：`/code/hermes-agent/agent/prompt_builder.py`，读 `SKILLS_GUIDANCE` 和 `build_skills_system_prompt()` 的 mandatory skills block。[锚点：`/code/hermes-agent/agent/prompt_builder.py:179`、`/code/hermes-agent/agent/prompt_builder.py:1175`]
6. Hermes read path：`/code/hermes-agent/tools/skills_tool.py`，读 `skills_list()`、`skill_view()` 和 plugin qualified name dispatch。[锚点：`/code/hermes-agent/tools/skills_tool.py:674`、`/code/hermes-agent/tools/skills_tool.py:849`]
7. Hermes write path：`/code/hermes-agent/tools/skill_manager_tool.py`，读 `_create_skill()`、`_patch_skill()` 和 `skill_manage()` schema。[锚点：`/code/hermes-agent/tools/skill_manager_tool.py:373`、`/code/hermes-agent/tools/skill_manager_tool.py:463`、`/code/hermes-agent/tools/skill_manager_tool.py:713`]
8. Hermes lifecycle：`/code/hermes-agent/tools/skill_usage.py` 和 `/code/hermes-agent/agent/curator.py`，读 `.usage.json`、agent-created provenance、state transitions、curator prompt。[锚点：`/code/hermes-agent/tools/skill_usage.py:1`、`/code/hermes-agent/agent/curator.py:256`、`/code/hermes-agent/agent/curator.py:330`]
9. Hermes hub：`/code/hermes-agent/tools/skills_hub.py`，读 `SkillSource`、hub state paths 和 lock/quarantine/audit 结构。[锚点：`/code/hermes-agent/tools/skills_hub.py:48`、`/code/hermes-agent/tools/skills_hub.py:294`]

## 最小复现抽象

以下是学习抽象，不是 OpenClaw 或 Hermes 的精确实现。

- `SkillRegistry`：维护已发现 skill 的 metadata、来源、优先级、可见性和状态；OpenClaw 可映射到 merged skill entries/snapshot，Hermes 可映射到 `skills_list()`、`skills_hub` lock 和 `.usage.json`。[学习抽象；锚点：`/code/openclaw/src/agents/skills/workspace.ts:768`、`/code/hermes-agent/tools/skills_tool.py:674`、`/code/hermes-agent/tools/skill_usage.py:1`]
- `SkillLoader`：按 root 优先级扫描 `SKILL.md`，处理 frontmatter、平台/工具条件、prompt injection/path traversal 安全检查，并提供 progressive disclosure 的 `view()`。[学习抽象；锚点：`/code/openclaw/src/agents/skills/workspace.ts:712`、`/code/hermes-agent/tools/skills_tool.py:849`]
- `SkillUsageTracker`：记录 view/use/patch 次数和最近活动时间，作为维护任务的输入；Hermes 有 `.usage.json` 明确实现，OpenClaw 本章未核实到同等 sidecar。[学习抽象；锚点：`/code/hermes-agent/tools/skill_usage.py:120`、`/code/hermes-agent/tools/skill_usage.py:405`]
- `SkillWriter`：创建、patch、写 supporting files、删除/归档 skill，并做 frontmatter/content/security validation；Hermes 的 `skill_manage()` 是直接样板，OpenClaw 的 Skill Workshop 是插件化写入入口。[学习抽象；锚点：`/code/hermes-agent/tools/skill_manager_tool.py:713`、`/code/openclaw/docs/tools/skills.md:106`]
- `SkillCurator`：根据 inactivity、usage、staleness 和 umbrella consolidation 策略维护 agent-created skills；Hermes 有 curator 明确实现，OpenClaw 本章只抽象到 workshop/ClawHub/update，不写成有 curator。[学习抽象；锚点：`/code/hermes-agent/agent/curator.py:1`、`/code/openclaw/docs/tools/skills.md:123`]

## 容易误解的点

- 不要把 OpenClaw plugin 等同于 skill：plugin 是能力/ownership 边界，skill 是可随 plugin 携带的操作指南，且 plugin skills 以低优先级 extra-dir 方式进入 skill loader。[锚点：`/code/openclaw/docs/plugins/architecture.md:31`、`/code/openclaw/docs/tools/skills.md:89`]
- 不要以为 OpenClaw 的 `$CODEX_HOME/skills` 会自动加载；文档明确说它不是 OpenClaw skill roots。[锚点：`/code/openclaw/docs/tools/skills.md:32`]
- 不要以为 OpenClaw extraDirs 可以覆盖 workspace skills；源码 merge 顺序和文档都显示 workspace 优先级最高，extra 最低。[锚点：`/code/openclaw/src/agents/skills/workspace.ts:768`、`/code/openclaw/docs/tools/skills.md:17`]
- 不要把 Hermes memory 当作 procedures 的存放地；`MEMORY_GUIDANCE` 明确 procedures/workflows belong in skills。[锚点：`/code/hermes-agent/agent/prompt_builder.py:150`]
- 不要把 Hermes curator 理解为会删除任意 skill；curator invariant 写明只 touching agent-created skills 且 never auto-deletes，只 archive。[锚点：`/code/hermes-agent/agent/curator.py:15`]
- 不要把 `skill_manage(create)` 都视为 curator-managed；源码注释说明只有 background self-improvement review fork 创建的 skill 才会 `mark_agent_created()`，foreground create 属于 user-directed。[锚点：`/code/hermes-agent/tools/skill_manager_tool.py:771`]

## 待核实问题

- OpenClaw Skill Workshop 的具体插件实现、approval/quarantine 流程和 snapshot refresh 调用路径未逐行核实，本章只引用 docs 声明。[锚点：`/code/openclaw/docs/tools/skills.md:106`]
- OpenClaw ClawHub install/update CLI 的源码实现未展开，本章只核实到 docs 中 install/update 行为和 security scan 声明。[锚点：`/code/openclaw/docs/tools/skills.md:123`、`/code/openclaw/docs/tools/skills.md:145`]
- Hermes curator 的 `maybe_run_curator()` 具体调用点、idle 判定来源和 forked `AIAgent` 参数未完整展开，本章主要依据 curator module 顶部说明和 state transition 函数。[锚点：`/code/hermes-agent/agent/curator.py:1`、`/code/hermes-agent/agent/curator.py:199`]
- Hermes plugin-provided skills 的 plugin manager 注册路径未完整追踪，本章只核实 `skill_view()` 的 qualified dispatch 和 serving path。[锚点：`/code/hermes-agent/tools/skills_tool.py:872`]
