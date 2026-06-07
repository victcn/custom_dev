# Long-Lived Agent Study Booklet Design

## Purpose

Create a mechanism-first study booklet for understanding long-lived agent systems through two local repositories:

- OpenClaw: `/code/openclaw`, reference commit `ca31a705d02e42ffcfb2c5884bb55339a6d0cbdc`
- Hermes Agent: `/code/hermes-agent`, reference commit `3d4297a59a8607ed24850524d229f5f42520d087`

The booklet is for learning and later recall. It should explain principles, guide source reading, compare similarities and differences, and extract minimal reproduction ideas. It is not an implementation plan for a new agent yet.

## Output Location

All study material should live under:

```text
/code/code-dev/agent_dev/docs/agent-lifecycle-study/
```

The main design file is this document. The booklet chapters should be sibling Markdown files in the same directory.

## Organization

Use a mechanism-first structure. Each chapter covers one core mechanism of a long-lived agent and compares OpenClaw and Hermes inside that topic.

Planned chapters:

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

Chapter scope:

- `00-reading-guide.md`: how to use the booklet, repository versions, terminology, and recommended reading order.
- `01-architecture-overview.md`: what a long-lived agent is, shared vocabulary, and the high-level difference between OpenClaw and Hermes.
- `02-gateway.md`: daemon/process model, platform adapters, WebSocket/RPC or gateway APIs, routing, delivery, pairing, and safety boundaries.
- `03-agent-loop.md`: message intake, context assembly, model call, tool execution, streaming, finalization, and persistence.
- `04-session-queue.md`: session identity, queues, lanes, write locks, steering, interruption, and transcript state.
- `05-memory.md`: long-term memory, short-term notes, provider orchestration, recall, compaction, dreaming, and user modeling.
- `06-skills.md`: procedural memory, skill discovery/loading, skill creation, maintenance, curator behavior, and plugin overlap.
- `07-automation.md`: cron, heartbeat, commitments, background tasks, scheduled runs, and delivery.
- `08-tools-plugins.md`: tool registry, hooks, model providers, plugin systems, safety scans, and approvals.
- `09-multi-agent.md`: subagents, isolated workspaces, delegation, routing, kanban/task lanes, and follow-up behavior.
- `10-failure-modes.md`: timeouts, retries, stuck sessions, prompt injection, sandboxing, and recovery.
- `11-minimal-reproduction.md`: a small architecture distilled from the two systems, clearly marked as an abstraction rather than a claim about either implementation.

## Accuracy Rules

The booklet must be evidence-first.

- Each chapter should cite the repository paths and commit references it relies on.
- Mechanism claims should include source or documentation anchors such as file paths, function names, class names, or docs pages.
- Separate three kinds of statements:
  - Source fact: directly observed in code.
  - Documentation claim: stated in project docs or README.
  - Study abstraction: the author's synthesis for learning or minimal reproduction.
- Do not present unverified guesses as facts.
- Put unresolved or partially verified items in `待核实问题`.
- Comparison tables should only compare mechanisms that were actually inspected.
- The minimal reproduction chapter must say it is an extracted design model, not the exact architecture of OpenClaw or Hermes.

## Per-Chapter Template

Each chapter should use this shape unless a chapter has a strong reason to differ:

```markdown
# <Chapter Title>

## 本章问题

## OpenClaw 怎么做

## Hermes 怎么做

## 异同点表

## 源码阅读路线

## 最小复现抽象

## 容易误解的点

## 待核实问题
```

Section intent:

- `本章问题`: explain the long-lifecycle problem this mechanism solves.
- `OpenClaw 怎么做`: architecture explanation plus key docs/source paths.
- `Hermes 怎么做`: architecture explanation plus key docs/source paths.
- `异同点表`: concise evidence-backed comparison.
- `源码阅读路线`: ordered files, functions, and tests to read.
- `最小复现抽象`: smallest components/interfaces needed to reproduce the mechanism later.
- `容易误解的点`: clarify misleading similarities or names.
- `待核实问题`: list incomplete areas instead of hiding uncertainty.

## Initial Evidence Map

OpenClaw starting anchors:

- `README.md`: product framing, install, gateway, channels, tools, security defaults.
- `docs/concepts/architecture.md`: Gateway architecture and WebSocket protocol summary.
- `docs/concepts/agent.md`: embedded runtime, workspace, bootstrap files, sessions.
- `docs/concepts/agent-loop.md`: authoritative agent loop lifecycle.
- `docs/concepts/memory.md`: memory files, search, compaction flush, dreaming.
- `src/agents/pi-embedded.ts`
- `src/agents/pi-embedded-runner/*.ts`
- `src/process/command-queue.ts`
- `src/config/sessions/*.ts`
- `src/hooks/plugin-hooks.ts`
- `packages/memory-host-sdk/src/**`
- `src/cron/isolated-agent/**`

Hermes starting anchors:

- `README.md` and `README.zh-CN.md`: project framing, self-improving loop, gateway, skills, memory, cron, subagents.
- `run_agent.py`: `AIAgent`, prompt building, memory manager integration, tool loop, `run_conversation`.
- `agent/prompt_builder.py`: identity, memory guidance, skills guidance, kanban guidance, context-file scanning.
- `agent/memory_manager.py`: memory provider orchestration, fenced memory context, prefetch/sync/tool routing.
- `agent/curator.py`: background skill maintenance.
- `gateway/session.py`: session context, routing context, dynamic system prompt injection.
- `cron/scheduler.py`: scheduled jobs, skill loading, prompt scanning, isolated cron runs.
- `providers/README.md`: provider registry and provider profile contract.
- `tools/skills_tool.py`, `tools/skills_hub.py`, `tools/registry.py`
- `environments/**`: benchmark/research-oriented agent environments.

## Design Decisions

- Use Chinese prose for the main explanations, because the user asked in Chinese and wants learning material.
- Keep file paths and symbol names in their original language.
- Prefer readable tutorial prose over dense reference tables, but include tables where they improve recall.
- Avoid implementation work in this phase. The next phase is writing the booklet chapters.

## Completion Criteria

The study booklet is considered complete for the first pass when:

- All planned chapter files exist.
- Every chapter has OpenClaw and Hermes sections.
- Every factual mechanism claim has at least one source path or documentation anchor.
- Every chapter has an `异同点表` and `源码阅读路线`.
- `11-minimal-reproduction.md` clearly labels abstractions and does not blur them with source facts.
- Unverified items are listed under `待核实问题`.
