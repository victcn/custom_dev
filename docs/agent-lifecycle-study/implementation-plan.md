# 长生命周期 Agent 学习小册子 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 写出一套中文机制优先小册子，准确对照 OpenClaw 和 Hermes Agent 的长生命周期 agent 原理、源码路径、异同点和最小复现抽象，并把 Codex / Claude Code 作为官方文档参照纳入入口、SDK/MCP/tool-use、memory、skills、automation、subagents 和 failure policy 边界。

**Architecture:** 小册子按机制拆章，每章固定包含问题、OpenClaw 做法、Hermes 做法、Codex/Claude Code 官方参照、异同点、参照表、源码阅读路线、最小复现抽象、误解点和待核实问题。所有事实性结论必须附源码或文档锚点；抽象性归纳必须明确标注为学习抽象。

**Tech Stack:** Markdown 文档、`rg`/`sed`/`find` 源码检索、Git commit 引用、中文教程写作。

---

## 文件结构

- 已存在：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/design.md`
  - 设计规格、语言要求、章节边界、准确性规则。
- 创建：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/00-reading-guide.md`
  - 阅读方法、版本、术语、推荐顺序。
- 创建：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/01-architecture-overview.md`
  - 总体架构、OpenClaw/Hermes 定位差异，以及 Codex/Claude Code 入口/扩展面参照。
- 创建：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/02-gateway.md`
  - Gateway 机制专章。
- 创建：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/03-agent-loop.md`
  - 主 agent loop 机制。
- 创建：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/04-session-queue.md`
  - session、queue、lane、锁和 transcript。
- 创建：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/05-memory.md`
  - 记忆机制。
- 创建：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/06-skills.md`
  - skill/过程记忆机制。
- 创建：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/07-automation.md`
  - cron、heartbeat、后台任务和投递。
- 创建：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/08-tools-plugins.md`
  - tools、plugins、hooks、provider、安全扫描。
- 创建：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/09-multi-agent.md`
  - 子代理、路由、隔离、kanban/task lanes。
- 创建：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/10-failure-modes.md`
  - 失败模式与恢复机制。
- 创建：`/code/code-dev/agent_dev/docs/agent-lifecycle-study/11-minimal-reproduction.md`
  - 从 OpenClaw/Hermes 源码事实和 Codex/Claude Code 官方参照中抽象出的最小长生命周期 agent 模型。

## 通用章节模板

每个章节文件使用以下结构：

```markdown
# Gateway

> 参考版本：
> - OpenClaw：`/code/openclaw@ca31a705d02e42ffcfb2c5884bb55339a6d0cbdc`
> - Hermes Agent：`/code/hermes-agent@3d4297a59a8607ed24850524d229f5f42520d087`
> - Codex / Claude Code：按章节需要列出官方文档参照。

## 本章问题

## OpenClaw 怎么做

## Hermes 怎么做

## Codex / Claude Code 参考

## 异同点表

## Codex / Claude Code 参照表

## 源码阅读路线

## 最小复现抽象

## 容易误解的点

## 待核实问题
```

上面示例标题为 `Gateway`；实际创建各章时，第一行必须替换为对应章节的中文标题，例如 `# 阅读指南`、`# 架构总览`、`# Agent Loop`。

## Task 1: 创建章节骨架和证据索引

**Files:**
- Create: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/00-reading-guide.md`
- Create: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/01-architecture-overview.md`
- Create: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/02-gateway.md`
- Create: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/03-agent-loop.md`
- Create: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/04-session-queue.md`
- Create: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/05-memory.md`
- Create: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/06-skills.md`
- Create: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/07-automation.md`
- Create: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/08-tools-plugins.md`
- Create: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/09-multi-agent.md`
- Create: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/10-failure-modes.md`
- Create: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/11-minimal-reproduction.md`

- [ ] **Step 1: 写入 12 个章节文件骨架**

使用通用章节模板创建所有章节。每章先只写标题、参考版本和固定二级标题。

- [ ] **Step 2: 记录每章初始证据锚点**

在每章的 `源码阅读路线` 下先放入本章需要核验的初始路径。第一批锚点：

```text
00-reading-guide.md:
- /code/openclaw/README.md
- /code/hermes-agent/README.md
- /code/hermes-agent/README.zh-CN.md
- /code/code-dev/agent_dev/docs/agent-lifecycle-study/design.md

01-architecture-overview.md:
- /code/openclaw/README.md
- /code/openclaw/docs/concepts/architecture.md
- /code/openclaw/docs/concepts/agent.md
- /code/hermes-agent/README.md
- /code/hermes-agent/providers/README.md

02-gateway.md:
- /code/openclaw/docs/concepts/architecture.md
- /code/openclaw/docs/gateway/protocol.md
- /code/hermes-agent/gateway/run.py
- /code/hermes-agent/gateway/session.py
- /code/hermes-agent/gateway/platforms/base.py

03-agent-loop.md:
- /code/openclaw/docs/concepts/agent-loop.md
- /code/openclaw/src/agents/pi-embedded.ts
- /code/openclaw/src/agents/pi-embedded-runner/run.ts
- /code/hermes-agent/run_agent.py

04-session-queue.md:
- /code/openclaw/docs/concepts/queue.md
- /code/openclaw/src/process/command-queue.ts
- /code/openclaw/src/config/sessions/store.ts
- /code/hermes-agent/gateway/session.py
- /code/hermes-agent/run_agent.py

05-memory.md:
- /code/openclaw/docs/concepts/memory.md
- /code/openclaw/packages/memory-host-sdk/src/engine.ts
- /code/hermes-agent/agent/memory_manager.py
- /code/hermes-agent/agent/memory_provider.py

06-skills.md:
- /code/openclaw/docs/tools/skills.md
- /code/openclaw/docs/plugins/architecture.md
- /code/hermes-agent/agent/curator.py
- /code/hermes-agent/tools/skills_tool.py
- /code/hermes-agent/tools/skills_hub.py

07-automation.md:
- /code/openclaw/docs/automation/cron-jobs.md
- /code/openclaw/docs/automation/heartbeat.md
- /code/openclaw/src/cron/isolated-agent/run.ts
- /code/hermes-agent/cron/scheduler.py
- /code/hermes-agent/cron/jobs.py

08-tools-plugins.md:
- /code/openclaw/src/hooks/plugin-hooks.ts
- /code/openclaw/docs/plugins/hooks.md
- /code/openclaw/docs/tools/index.md
- /code/hermes-agent/tools/registry.py
- /code/hermes-agent/providers/README.md

09-multi-agent.md:
- /code/openclaw/docs/concepts/multi-agent.md
- /code/openclaw/docs/concepts/delegate-architecture.md
- /code/hermes-agent/agent/prompt_builder.py
- /code/hermes-agent/docs/hermes-kanban-v1-spec.pdf

10-failure-modes.md:
- /code/openclaw/docs/concepts/retry.md
- /code/openclaw/docs/gateway/sandboxing.md
- /code/openclaw/docs/gateway/security/index.md
- /code/hermes-agent/run_agent.py
- /code/hermes-agent/cron/scheduler.py

11-minimal-reproduction.md:
- /code/code-dev/agent_dev/docs/agent-lifecycle-study/00-reading-guide.md
- /code/code-dev/agent_dev/docs/agent-lifecycle-study/01-architecture-overview.md
- /code/code-dev/agent_dev/docs/agent-lifecycle-study/02-gateway.md
- /code/code-dev/agent_dev/docs/agent-lifecycle-study/03-agent-loop.md
- /code/code-dev/agent_dev/docs/agent-lifecycle-study/04-session-queue.md
- /code/code-dev/agent_dev/docs/agent-lifecycle-study/05-memory.md
- /code/code-dev/agent_dev/docs/agent-lifecycle-study/06-skills.md
- /code/code-dev/agent_dev/docs/agent-lifecycle-study/07-automation.md
- /code/code-dev/agent_dev/docs/agent-lifecycle-study/08-tools-plugins.md
- /code/code-dev/agent_dev/docs/agent-lifecycle-study/09-multi-agent.md
- /code/code-dev/agent_dev/docs/agent-lifecycle-study/10-failure-modes.md
```

- [ ] **Step 3: 验证文件都存在**

Run:

```bash
find /code/code-dev/agent_dev/docs/agent-lifecycle-study -maxdepth 1 -type f | sort
```

Expected: 输出包含 `design.md`、`implementation-plan.md`、`00-reading-guide.md`、`01-architecture-overview.md`、`02-gateway.md`、`03-agent-loop.md`、`04-session-queue.md`、`05-memory.md`、`06-skills.md`、`07-automation.md`、`08-tools-plugins.md`、`09-multi-agent.md`、`10-failure-modes.md`、`11-minimal-reproduction.md`。

## Task 2: 写 `00-reading-guide.md` 和 `01-architecture-overview.md`

**Files:**
- Modify: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/00-reading-guide.md`
- Modify: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/01-architecture-overview.md`

- [ ] **Step 1: 读取总览证据**

Run:

```bash
sed -n '1,220p' /code/openclaw/README.md
sed -n '1,240p' /code/openclaw/docs/concepts/architecture.md
sed -n '1,220p' /code/openclaw/docs/concepts/agent.md
sed -n '1,240p' /code/hermes-agent/README.md
sed -n '1,220p' /code/hermes-agent/README.zh-CN.md
sed -n '1,220p' /code/hermes-agent/providers/README.md
```

Expected: 能定位 OpenClaw 的 Gateway/本地助理定位，以及 Hermes 的 self-improving/skills/memory/provider 定位。

- [ ] **Step 2: 写 `00-reading-guide.md` 正文**

必须包含：

- 仓库版本和路径。
- “源码事实 / 文档声明 / 学习抽象”的阅读标记。
- 推荐阅读顺序：先 `01`、`02`、`03`，再按需要读 `05`/`06`/`07`。
- 术语表：`Gateway`、`agent loop`、`session`、`memory`、`skill`、`tool`、`cron`、`subagent`。

- [ ] **Step 3: 写 `01-architecture-overview.md` 正文**

必须包含：

- 长生命周期 agent 的定义：长期运行入口、持久 session、可恢复状态、跨会话记忆、后台自动化、工具/插件边界。
- OpenClaw 总体模型：本地优先 Gateway + embedded agent runtime + workspace/session/memory/plugins。
- Hermes 总体模型：Python `AIAgent` + gateway/platforms + `MemoryManager` + skill curator + cron/subagents。
- 异同点表至少包含：定位、运行入口、状态持久化、记忆、技能、自动化、插件/provider。

- [ ] **Step 4: 自检引用锚点**

Run:

```bash
rg -n "/code/openclaw|/code/hermes-agent|源码事实|文档声明|学习抽象" /code/code-dev/agent_dev/docs/agent-lifecycle-study/00-reading-guide.md /code/code-dev/agent_dev/docs/agent-lifecycle-study/01-architecture-overview.md
```

Expected: 两章都包含仓库路径，且 `00-reading-guide.md` 解释三类表述。

## Task 3: 写 `02-gateway.md`

**Files:**
- Modify: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/02-gateway.md`

- [ ] **Step 1: 读取 Gateway 证据**

Run:

```bash
sed -n '1,260p' /code/openclaw/docs/concepts/architecture.md
sed -n '1,260p' /code/openclaw/docs/gateway/protocol.md
sed -n '1,260p' /code/hermes-agent/gateway/session.py
sed -n '1,220p' /code/hermes-agent/gateway/platforms/base.py
sed -n '1,220p' /code/hermes-agent/gateway/run.py
```

Expected: 能说明 OpenClaw 的 WebSocket Gateway、client/node connect、pairing、events；能说明 Hermes gateway 的 platform/session context/routing。

- [ ] **Step 2: 写 OpenClaw Gateway 部分**

必须覆盖：

- 单个长生命周期 Gateway daemon。
- WebSocket 控制面、请求/响应/event。
- clients 与 nodes 的角色差异。
- pairing/auth 的边界。
- Gateway 与 agent loop 的连接方式。

- [ ] **Step 3: 写 Hermes Gateway 部分**

必须覆盖：

- `gateway/session.py` 如何表示 `SessionSource` 和 `SessionContext`。
- platform adapter 的共同边界。
- gateway 如何为 agent 提供“消息来自哪里、应该投递到哪里”的上下文。

- [ ] **Step 4: 写异同点和最小复现抽象**

异同点表必须使用 `维度 | OpenClaw | Hermes | 学习结论` 四列，并覆盖这些行：`入口形态`、`协议/适配`、`会话上下文`、`安全边界`、`与 agent loop 的关系`。每个单元格只能写已核实结论；未核实内容写入 `待核实问题`。

最小复现抽象至少包含：

- `GatewayServer`
- `PlatformAdapter`
- `SessionRouter`
- `DeliveryTarget`
- `AuthPairingStore`

- [ ] **Step 5: 验证 Gateway 章节没有无证据断言**

Run:

```bash
rg -n "应该|可能|大概|猜测|待核实" /code/code-dev/agent_dev/docs/agent-lifecycle-study/02-gateway.md
```

Expected: 如果有推测，必须出现在 `待核实问题` 或明确标注为学习抽象。

## Task 4: 写 `03-agent-loop.md` 和 `04-session-queue.md`

**Files:**
- Modify: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/03-agent-loop.md`
- Modify: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/04-session-queue.md`

- [ ] **Step 1: 读取 agent loop 和 session 证据**

Run:

```bash
sed -n '1,320p' /code/openclaw/docs/concepts/agent-loop.md
sed -n '1,320p' /code/openclaw/src/agents/pi-embedded-runner/run.ts
sed -n '1,260p' /code/openclaw/src/agents/pi-embedded-runner/runs.ts
sed -n '1,260p' /code/openclaw/src/process/command-queue.ts
rg -n "class AIAgent|def run_conversation|def _build_system_prompt|handle_function_call|ContextCompressor|memory_manager" /code/hermes-agent/run_agent.py
sed -n '1028,1240p' /code/hermes-agent/run_agent.py
sed -n '11410,11920p' /code/hermes-agent/run_agent.py
```

Expected: 能定位 OpenClaw 的 `runEmbeddedPiAgent` 和 Hermes 的 `AIAgent.run_conversation`。

- [ ] **Step 2: 写 `03-agent-loop.md`**

必须覆盖：

- OpenClaw：`agent` RPC 接收、立即 ack、`agentCommand`、`runEmbeddedPiAgent`、stream bridge、tool/assistant/lifecycle 事件。
- Hermes：`AIAgent` 初始化、system prompt、messages、tool calls、tool result、context compression、final response。
- 对比：事件驱动/嵌入 runtime 与 Python 单体 loop 的差异。
- 最小复现抽象：`AgentRun`, `PromptBuilder`, `ModelClient`, `ToolExecutor`, `StreamEmitter`, `RunFinalizer`。

- [ ] **Step 3: 写 `04-session-queue.md`**

必须覆盖：

- OpenClaw：session key、per-session lane、global lane、queue modes、transcript write lock。
- Hermes：session context、conversation/session persistence、gateway routing context、interrupt/redirect 相关行为只写已核实部分。
- 对比：OpenClaw 更强调 Gateway/session lane 一致性；Hermes 更强调运行时消息历史和平台上下文。
- 最小复现抽象：`SessionId`, `SessionStore`, `SessionLane`, `RunQueue`, `TranscriptLock`, `InterruptController`。

- [ ] **Step 4: 验证核心符号出现**

Run:

```bash
rg -n "runEmbeddedPiAgent|agentCommand|AIAgent|run_conversation|Session|queue|lane|transcript" /code/code-dev/agent_dev/docs/agent-lifecycle-study/03-agent-loop.md /code/code-dev/agent_dev/docs/agent-lifecycle-study/04-session-queue.md
```

Expected: 两章都能通过符号回到源码。

## Task 5: 写 `05-memory.md` 和 `06-skills.md`

**Files:**
- Modify: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/05-memory.md`
- Modify: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/06-skills.md`

- [ ] **Step 1: 读取 memory 和 skills 证据**

Run:

```bash
sed -n '1,280p' /code/openclaw/docs/concepts/memory.md
sed -n '1,260p' /code/openclaw/docs/tools/skills.md
sed -n '1,260p' /code/openclaw/docs/plugins/architecture.md
sed -n '1,320p' /code/hermes-agent/agent/memory_manager.py
sed -n '1,320p' /code/hermes-agent/agent/curator.py
sed -n '1,260p' /code/hermes-agent/agent/prompt_builder.py
sed -n '1,260p' /code/hermes-agent/tools/skills_tool.py
```

Expected: 能定位 OpenClaw 的 memory files/search/dreaming/compaction flush，以及 Hermes 的 `MemoryManager`/curator/skill guidance。

- [ ] **Step 2: 写 `05-memory.md`**

必须覆盖：

- OpenClaw：`MEMORY.md`、`memory/YYYY-MM-DD.md`、`DREAMS.md`、memory tools、memory backends、compaction 前自动 flush、dreaming。
- Hermes：`MemoryManager` provider 编排、`build_memory_context_block`、`prefetch_all`、`sync_all`、memory tool routing、memory guidance。
- 对比：文件/插件型 memory 与 provider manager 型 memory 的差异。
- 最小复现抽象：`MemoryStore`, `MemoryRetriever`, `MemoryInjector`, `MemoryWriter`, `MemoryMaintenanceJob`。

- [ ] **Step 3: 写 `06-skills.md`**

必须覆盖：

- OpenClaw：skills 加载位置、workspace/personal/managed/bundled/extraDirs，skills 与 plugins 的边界。
- Hermes：`SKILLS_GUIDANCE`、skill creation/update、`curator` 的 inactivity-triggered maintenance、skill lifecycle。
- 对比：OpenClaw 更偏工具/插件生态集成，Hermes 更突出从经验中维护 procedural memory。
- 最小复现抽象：`SkillRegistry`, `SkillLoader`, `SkillUsageTracker`, `SkillWriter`, `SkillCurator`。

- [ ] **Step 4: 验证 memory/skill 章节包含事实和抽象分界**

Run:

```bash
rg -n "源码事实|文档声明|学习抽象|MemoryManager|curator|MEMORY.md|DREAMS.md|skills" /code/code-dev/agent_dev/docs/agent-lifecycle-study/05-memory.md /code/code-dev/agent_dev/docs/agent-lifecycle-study/06-skills.md
```

Expected: 两章都包含源码或文档锚点，并且抽象内容有明确标识。

## Task 6: 写 `07-automation.md`、`08-tools-plugins.md`、`09-multi-agent.md`

**Files:**
- Modify: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/07-automation.md`
- Modify: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/08-tools-plugins.md`
- Modify: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/09-multi-agent.md`

- [ ] **Step 1: 读取 automation/tools/multi-agent 证据**

Run:

```bash
sed -n '1,260p' /code/openclaw/docs/automation/cron-jobs.md
sed -n '1,260p' /code/openclaw/docs/gateway/heartbeat.md
sed -n '1,260p' /code/openclaw/src/cron/isolated-agent/run.ts
sed -n '1,320p' /code/hermes-agent/cron/scheduler.py
sed -n '1,260p' /code/hermes-agent/cron/jobs.py
sed -n '1,260p' /code/openclaw/src/hooks/plugin-hooks.ts
sed -n '1,260p' /code/openclaw/docs/plugins/hooks.md
sed -n '1,260p' /code/hermes-agent/tools/registry.py
sed -n '1,260p' /code/hermes-agent/providers/README.md
sed -n '1,260p' /code/openclaw/docs/concepts/multi-agent.md
sed -n '1,260p' /code/openclaw/docs/concepts/delegate-architecture.md
rg -n "KANBAN_GUIDANCE|delegate|subagent|kanban" /code/hermes-agent/agent/prompt_builder.py /code/hermes-agent/run_agent.py
```

Expected: 能分别定位自动化、工具插件、多代理的可靠证据。

- [ ] **Step 2: 写 `07-automation.md`**

必须覆盖：

- OpenClaw：cron/heartbeat/commitments 的文档声明和 isolated agent run。
- Hermes：`cron/scheduler.py` 的 prompt assembly、skill loading、cron injection scanning、短生命周期 worker。
- 对比：自动化如何触发 agent run，以及如何投递结果。
- 最小复现抽象：`Scheduler`, `JobStore`, `PromptAssembler`, `RunLauncher`, `DeliveryDispatcher`。

- [ ] **Step 3: 写 `08-tools-plugins.md`**

必须覆盖：

- OpenClaw：plugin hooks、tool lifecycle hooks、provider plugins、gateway pipeline hooks。
- Hermes：`tools/registry.py`、provider registry、skill/tool safety scanning。
- 对比：OpenClaw 插件边界更系统化，Hermes provider/tools/skills 分布在 Python registry 与插件目录中。
- 最小复现抽象：`ToolRegistry`, `PluginRegistry`, `HookBus`, `ProviderRegistry`, `SafetyScanner`。

- [ ] **Step 4: 写 `09-multi-agent.md`**

必须覆盖：

- OpenClaw：multi-agent routing、workspace/session isolation、delegate architecture。
- Hermes：subagent/delegate/kanban guidance 的已核实部分。
- 对比：路由隔离型 multi-agent 与任务委派/kanban 型 multi-agent 的差异。
- 最小复现抽象：`AgentProfile`, `WorkspaceAllocator`, `DelegationQueue`, `TaskBoard`, `SubagentRunner`。

- [ ] **Step 5: 验证三章没有把未读 PDF 写成事实**

Run:

```bash
rg -n "hermes-kanban-v1-spec|PDF|待核实|学习抽象" /code/code-dev/agent_dev/docs/agent-lifecycle-study/09-multi-agent.md
```

Expected: 如果没有实际解析 `docs/hermes-kanban-v1-spec.pdf`，相关内容只能在 `待核实问题` 或学习抽象中出现。

## Task 7: 写 `10-failure-modes.md` 和 `11-minimal-reproduction.md`

**Files:**
- Modify: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/10-failure-modes.md`
- Modify: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/11-minimal-reproduction.md`

- [ ] **Step 1: 读取失败模式证据**

Run:

```bash
sed -n '1,260p' /code/openclaw/docs/concepts/retry.md
sed -n '1,280p' /code/openclaw/docs/gateway/sandboxing.md
sed -n '1,260p' /code/openclaw/docs/gateway/security/index.md
rg -n "timeout|retry|stuck|abort|sandbox|prompt injection|approval" /code/openclaw/docs /code/openclaw/src | head -200
rg -n "timeout|retry|stuck|abort|prompt injection|approval|sandbox" /code/hermes-agent/run_agent.py /code/hermes-agent/cron/scheduler.py /code/hermes-agent/agent /code/hermes-agent/tools | head -220
```

Expected: 能提取两项目关于 timeout、retry、sandbox/security、prompt injection scan 的证据。

- [ ] **Step 2: 写 `10-failure-modes.md`**

必须覆盖：

- OpenClaw：agent timeout、model idle timeout、stuck session diagnostics、sandboxing、安全默认值。
- Hermes：API/tool call repair、cron prompt injection scanning、timeouts、safe writer、provider retry。
- 对比：长期运行系统如何防止 run 卡死、状态损坏、越权工具调用、上下文污染。
- 最小复现抽象：`TimeoutPolicy`, `RetryPolicy`, `AbortController`, `SandboxPolicy`, `PromptInjectionScanner`, `RecoveryMonitor`。

- [ ] **Step 3: 写 `11-minimal-reproduction.md`**

必须明确分为三层：

```text
第一层：最小可运行长期 agent
- Gateway/CLI 入口
- SessionStore
- AgentLoop
- ToolRegistry
- TranscriptStore

第二层：真正长生命周期所需能力
- MemoryStore + MemoryRetriever
- Scheduler
- DeliveryTarget
- Queue/Lane
- FailureRecovery

第三层：接近 OpenClaw/Hermes，并吸收 Codex/Claude Code 外部参照的高级能力
- Plugin/Hook system
- Skill curator
- Multi-agent routing/delegation
- Sandboxed workspaces
- Provider registry/failover
```

必须说明这些是学习抽象，不代表 OpenClaw、Hermes、Codex 或 Claude Code 的精确实现。

- [ ] **Step 4: 验证最小复现章节的边界声明**

Run:

```bash
rg -n "学习抽象|不是 OpenClaw|不是 Hermes|精确实现|最小" /code/code-dev/agent_dev/docs/agent-lifecycle-study/11-minimal-reproduction.md
```

Expected: 明确出现抽象边界声明，避免误读为源码事实。

## Task 8: 全书准确性和一致性审校

**Files:**
- Modify as needed: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/*.md`

- [ ] **Step 1: 检查章节完整性**

Run:

```bash
for f in /code/code-dev/agent_dev/docs/agent-lifecycle-study/{00-reading-guide,01-architecture-overview,02-gateway,03-agent-loop,04-session-queue,05-memory,06-skills,07-automation,08-tools-plugins,09-multi-agent,10-failure-modes,11-minimal-reproduction}.md; do
  echo "== $f =="
  rg -n "^## 本章问题|^## OpenClaw 怎么做|^## Hermes 怎么做|^## 异同点表|^## 源码阅读路线|^## 最小复现抽象|^## 容易误解的点|^## 待核实问题" "$f"
done
```

Expected: 每个章节都包含固定模板中的 8 个二级标题。

- [ ] **Step 2: 检查未完成标记和弱断言**

Run:

```bash
rg -n "TB[D]|TO[D]O|FIXM[E]|待[定]|以后[补]|这里[补]|大概|应该是|可能是|猜测" /code/code-dev/agent_dev/docs/agent-lifecycle-study/{00-reading-guide,01-architecture-overview,02-gateway,03-agent-loop,04-session-queue,05-memory,06-skills,07-automation,08-tools-plugins,09-multi-agent,10-failure-modes,11-minimal-reproduction}.md
```

Expected: 没有未完成标记。 如果出现“大概/应该是/可能是/猜测”，只能出现在 `待核实问题` 或明确标注为学习抽象的段落中。

- [ ] **Step 3: 检查每章都有路径锚点**

Run:

```bash
for f in /code/code-dev/agent_dev/docs/agent-lifecycle-study/{00-reading-guide,01-architecture-overview,02-gateway,03-agent-loop,04-session-queue,05-memory,06-skills,07-automation,08-tools-plugins,09-multi-agent,10-failure-modes,11-minimal-reproduction}.md; do
  echo "== $f =="
  rg -n "/code/openclaw|/code/hermes-agent|src/|docs/|agent/|gateway/|cron/|tools/|packages/" "$f" | head
done
```

Expected: 每章都至少有 OpenClaw 和 Hermes 的源码或文档路径；涉及入口、loop、session、memory、skills、automation、tools/MCP、subagents、failure policy 或最小复现的章节，还要有 Codex / Claude Code 官方文档参照。

- [ ] **Step 4: 检查语言要求**

Run:

```bash
rg -n "This chapter|Overview|Conclusion|Similarities|Differences|TO[D]O" /code/code-dev/agent_dev/docs/agent-lifecycle-study/{00-reading-guide,01-architecture-overview,02-gateway,03-agent-loop,04-session-queue,05-memory,06-skills,07-automation,08-tools-plugins,09-multi-agent,10-failure-modes,11-minimal-reproduction}.md
```

Expected: 不出现英文模板标题。文件路径、函数名、类名、命令、配置键可以保留英文。

- [ ] **Step 5: 输出最终文件列表**

Run:

```bash
find /code/code-dev/agent_dev/docs/agent-lifecycle-study -maxdepth 1 -type f -name '*.md' | sort
```

Expected: 输出 `design.md`、`implementation-plan.md` 和 12 个章节文件。

## Task 9: 最终交付说明

**Files:**
- Read: `/code/code-dev/agent_dev/docs/agent-lifecycle-study/*.md`

- [ ] **Step 1: 汇总完成情况**

最终回复用户时说明：

- 小册子路径。
- 已覆盖章节。
- 参考 commit。
- 哪些内容仍在 `待核实问题`。
- `/code/code-dev/agent_dev` 不是 git 仓库，因此没有 commit。

- [ ] **Step 2: 建议下一步**

给出两个下一步选项：

```text
1. 继续深入某一章，例如 Gateway、memory 或 skills。
2. 基于第 11 章，开始设计自己的长生命周期 agent MVP。
```
