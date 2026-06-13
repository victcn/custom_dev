# 长生命周期 Agent 学习小册子设计

## 目标

基于两个本地仓库，并补充 Codex / Claude Code 的官方产品文档参照，写一套“机制优先”的长生命周期 agent 学习小册子：

- OpenClaw：`/code/openclaw`，参考 commit `ca31a705d02e42ffcfb2c5884bb55339a6d0cbdc`
- Hermes Agent：`/code/hermes-agent`，参考 commit `3d4297a59a8607ed24850524d229f5f42520d087`

这套材料用于学习和后续复习。它需要讲清楚原理、引导源码阅读、比较 OpenClaw/Hermes 的本地源码事实，同时用 Codex/Claude Code 校准 CLI、SDK、MCP、tool-use、automation、memory、skills、subagents 等外部产品边界，并抽象出最小复现思路。本阶段不是为新的 agent 写实现计划。

Codex 和 Claude Code 只作为官方文档参照，不作为本小册子的逐章源码对照对象；源码事实主线仍以 OpenClaw 和 Hermes Agent 为准。涉及 Codex/Claude Code 的段落必须明确写成“官方文档参照”或“外部产品参照”，不能写成本地源码事实。

## 语言要求

后续所有学习材料默认使用中文，包括正文解释、章节标题、表格说明、异同点总结和最小复现抽象。

保留原文的内容：

- 文件路径
- 函数名、类名、变量名
- 命令
- 配置键
- 项目自己的术语，例如 `Gateway`、`agent loop`、`MemoryManager`、`curator`

如果英文术语有必要解释，第一次出现时使用“英文术语 + 中文解释”的形式。

## 输出位置

所有学习材料放在：

```text
/code/code-dev/agent_dev/docs/agent-lifecycle-study/
```

本文档是设计规格。正式小册子的章节文件与本文档位于同一目录。

## 组织方式

采用“机制优先”的结构。每章讲一个长生命周期 agent 的核心机制，并在同一章内对照 OpenClaw 和 Hermes 的做法；凡是机制与入口、session、agent loop、memory、skills、automation、tools/MCP、subagents 或 failure policy 相关，都要补充 Codex/Claude Code 的官方文档参照，用来校准 CLI、SDK、MCP、API tool use、hooks 与 Gateway 的边界。

计划章节：

```text
00-reading-guide.md
01-architecture-overview.md
02-gateway.md
03-agent-loop.md
04-session-queue.md
05-memory.md
06-skills.md
07-automation.md
08-tools-plugins.md
09-multi-agent.md
10-failure-modes.md
11-minimal-reproduction.md
```

章节边界：

- `00-reading-guide.md`：如何使用这套材料、仓库版本、术语表、推荐阅读顺序。
- `01-architecture-overview.md`：什么是长生命周期 agent、共用词汇，以及 OpenClaw/Hermes 的总体差异；补充 Codex/Claude Code 作为入口、扩展面和宿主机制参照。
- `02-gateway.md`：daemon/process 模型、平台适配器、WebSocket/RPC、HTTP API、Webhook、CLI/SDK/MCP 入口、路由、投递、配对和安全边界。
- `03-agent-loop.md`：消息进入、上下文组装、模型调用、工具执行、流式输出、收尾和持久化。
- `04-session-queue.md`：session 身份、队列、lane、写锁、steering、打断和 transcript 状态。
- `05-memory.md`：长期记忆、短期笔记、memory provider、召回、compaction、dreaming 和 user modeling。
- `06-skills.md`：作为过程记忆的 skill，skill 发现/加载/创建/维护，Hermes curator，以及 OpenClaw skills/plugins 的关系。
- `07-automation.md`：cron、heartbeat、commitments、后台任务、定时运行和投递。
- `08-tools-plugins.md`：工具注册、hooks、模型 provider、插件系统、MCP/API tool use、安全扫描和审批。
- `09-multi-agent.md`：subagents、隔离工作区、委派、路由、kanban/task lanes 和 follow-up 行为。
- `10-failure-modes.md`：超时、重试、stuck session、prompt injection、sandboxing 和恢复策略。
- `11-minimal-reproduction.md`：从 OpenClaw/Hermes 的源码证据和 Codex/Claude Code 的官方产品参照中抽象出一个最小长生命周期 agent 架构，并明确它是学习抽象，不是任一系统真实实现的等同描述。

## 准确性规则

这套小册子必须证据优先。

- 每章标明依赖的仓库路径和 commit。
- 每个机制结论都尽量附源码或文档锚点，例如文件路径、函数名、类名或文档页。
- 明确区分三类表述：
  - 源码事实：直接从代码观察到。
  - 文档声明：项目 README 或 docs 中明确写到。
  - 学习抽象：为了理解或最小复现而做的归纳。
- 不把未验证猜测写成事实。
- 未解决或只部分验证的内容放进 `待核实问题`。
- 异同点表只比较已经实际查看过的机制。
- 最小复现章节必须说明它是学习抽象，不是 OpenClaw、Hermes、Codex 或 Claude Code 的精确实现；其中 Codex/Claude Code 只能用于官方文档可核验的入口、SDK/MCP/tool-use、memory、skills、automation、subagents、hooks/failure policy 参照。

## 每章模板

除非某章有强理由需要调整，否则每章使用以下结构：

```markdown
# <章节标题>

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

各部分用途：

- `本章问题`：解释这个机制解决了长生命周期 agent 的什么问题。
- `OpenClaw 怎么做`：架构解释 + 关键文档/源码路径。
- `Hermes 怎么做`：架构解释 + 关键文档/源码路径。
- `Codex / Claude Code 参考`：只写官方文档中能核验的入口、SDK/MCP/tool-use、memory、skills、automation、subagents 或 policy 边界；不写私有源码事实。
- `异同点表`：只写 OpenClaw/Hermes 本地源码或本地文档证据支撑的比较。
- `Codex / Claude Code 参照表`：把同一机制下 Codex/Claude Code 的官方产品参照单独成表，避免和 OpenClaw/Hermes 源码事实混列。
- `源码阅读路线`：按顺序列出建议阅读的文件、函数和测试。
- `最小复现抽象`：如果以后自己实现，最少需要哪些组件和接口。
- `容易误解的点`：澄清“看起来相似但实际不同”的地方。
- `待核实问题`：记录未读透或需要继续验证的点。

## 初始证据地图

OpenClaw 起始锚点：

- `README.md`：产品定位、安装、gateway、channels、tools、安全默认值。
- `docs/concepts/architecture.md`：Gateway 架构和 WebSocket 协议概览。
- `docs/concepts/agent.md`：embedded runtime、workspace、bootstrap files、sessions。
- `docs/concepts/agent-loop.md`：agent loop 生命周期。
- `docs/concepts/memory.md`：memory files、search、compaction flush、dreaming。
- `src/agents/pi-embedded.ts`
- `src/agents/pi-embedded-runner/*.ts`
- `src/process/command-queue.ts`
- `src/config/sessions/*.ts`
- `src/hooks/plugin-hooks.ts`
- `packages/memory-host-sdk/src/**`
- `src/cron/isolated-agent/**`

Hermes 起始锚点：

- `README.md` 和 `README.zh-CN.md`：项目定位、自我改进闭环、gateway、skills、memory、cron、subagents。
- `run_agent.py`：`AIAgent`、prompt 构建、memory manager 集成、tool loop、`run_conversation`。
- `agent/prompt_builder.py`：identity、memory guidance、skills guidance、kanban guidance、context-file scanning。
- `agent/memory_manager.py`：memory provider 编排、fenced memory context、prefetch/sync/tool routing。
- `agent/curator.py`：后台 skill 维护。
- `gateway/session.py`：session context、路由上下文、动态 system prompt 注入。
- `cron/scheduler.py`：定时任务、skill 加载、prompt 扫描、隔离 cron run。
- `providers/README.md`：provider registry 和 provider profile contract。
- `tools/skills_tool.py`、`tools/skills_hub.py`、`tools/registry.py`
- `environments/**`：benchmark/research-oriented agent environments。

## 设计决策

- 主体说明使用中文，满足学习和复习需求。
- 文件路径、符号名、命令和配置键保留原文，避免失真。
- 优先写成可读教程，而不是密集 reference；但在异同点、源码路线、复习要点处使用表格。
- 本阶段不做实现工作。下一阶段是规划并撰写小册子章节。

## 第一版完成标准

第一版小册子满足以下条件才算完成：

- 所有计划章节文件都存在。
- 每章都有 OpenClaw 和 Hermes 两部分。
- 每个事实性机制结论至少有一个源码路径或文档锚点。
- 每章都有 `异同点表` 和 `源码阅读路线`。
- `11-minimal-reproduction.md` 明确标注哪些内容是学习抽象，不与源码事实混淆。
- 未验证内容全部列在 `待核实问题`。
