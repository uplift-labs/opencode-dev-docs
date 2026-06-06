# Orchestrator Design

Design-only brief for a future OpenCode orchestration product. Recommended package name: `opencode-orchestrator`. Previous working names: `multitask`, `conductor`; this file keeps the old filename for link stability.

Goal: one master OpenCode session decomposes a large task, chooses the minimum useful isolation for each worker, dispatches work in parallel, tracks status, and aggregates reports back into the master session. Isolation escalates only when needed: same-session subagent UX first, then child sessions in the same workspace, then worktree-backed workspaces.

No implementation code belongs in this document. Use it as the source pack for a new repository.

Source note: OpenCode anchors were checked against local upstream checkout `18ba80f`; external package review was last checked on 2026-06-06 and can drift faster than OpenCode source APIs.

## Task Clarity

The task is clear.

Desired user-facing behavior:

1. User starts in a master session and asks for a large task to be parallelized.
2. `orchestrator` plans independent work packages with file scopes, dependencies, models/agents, and risk labels.
3. For each work package, `orchestrator` chooses an execution mode: same-session subagent UX, same-workspace child session, new-workspace child session, or multiple sessions inside a new workspace.
4. After user approval, `orchestrator` creates only the sessions/workspaces required by that plan.
5. The master session remains the coordinator: it shows status, handles blockers, collects reports, and decides when to merge or ask the user.
6. Each worker ends with a structured report: summary, changed files if any, tests run, blockers, isolation mode, and merge readiness.

Important translation: the plugin should not literally click the UI button `New workspace`. Runtime orchestration should use the same underlying OpenCode workspace/session APIs from server-side code. UI automation is fragile; TUI code should trigger orchestrator through prompt-mediated UX or an explicit supported bridge, not by simulating UI clicks.

## Existing Solutions Review

Exact match found: no.

Close matches worth reusing or studying:

| Package | What Exists | Useful For `orchestrator` | Gap |
| --- | --- | --- | --- |
| [`opencode-forge`](https://github.com/chriswritescode-dev/opencode-forge) | Plans, execution dialog, isolated worktree loops, fresh sessions, optional OpenCode workspace integration. | Best reference implementation for `experimental.workspace.create`, `session.create`, `promptAsync`, session rotation, worktree loop state, and workspace status handling. | Oriented around one iterative plan/loop workflow, not N sibling worker sessions managed by one master session. |
| [`opencode-swarm`](https://github.com/zaxbysauce/opencode-swarm) | Architect-led specialized agents, QA gates, evidence store, Lean Turbo lane planning. | Good reference for planning, risk gates, scope enforcement, evidence, anti-loop controls, and report aggregation patterns. | Code search shows `worktree_isolation` is documented/configured but marked not implemented; it is not an exact worktree/workspace fan-out plugin. |
| [`opencode-swarm-plugin`](https://github.com/joelhooks/swarm-tools) | Decomposes tasks, spawns workers, uses Hive tasks, Swarm Mail, file reservations, durable local coordination. | Good reference for durable coordination, message bus, file reservations, and task decomposition. | Workers are not clearly OpenCode workspace-backed sessions. |
| [`opencode-devcontainers`](https://github.com/athal7/opencode-devcontainers) | `/devcontainer`, `/worktree`, `/workspaces`, isolated branch workspaces, env/secrets copy, ports. | Good reference for environment/worktree setup and cleanup. | Targets/switches current session; does not create multiple child sessions and aggregate results. |
| [`opencode-worktree-manager`](https://github.com/Mumme-IT/opencode-worktree-manager) | Worktree create/list/switch/status/finish tools plus TUI sidebar. | Good reference for worktree management UX and status. | No master-worker session orchestration. |
| [`opencode-worktree`](https://github.com/arturosdg/opencode-worktree) | Standalone TUI for git worktrees and launching a coding tool. | Useful external launcher reference. | Not an OpenCode plugin and no master session aggregation. |
| [`@trismegis7us/openaide`](https://github.com/trismegis7us/aide) | CLI creates git worktree + tmux session + OpenCode agent from prompt/spec. | Very close external process model for parallel workspaces. | Not an OpenCode plugin or workspace API integration. |
| [`@a5c-ai/babysitter-opencode`](https://github.com/a5c-ai/babysitter/tree/main/plugins/babysitter-opencode) | Event-sourced orchestration plugin and in-turn iteration model. | Useful for deterministic orchestration and human approval patterns. | No claimed worktree/workspace fan-out. |
| [`opencode-subagent-statusline`](https://github.com/Joaquinvesapa/sub-agent-statusline) | TUI sidebar for subagent status. | Useful visibility pattern for worker status. | Monitoring only. |

Recommendation: do not build a large swarm framework from scratch. Build `orchestrator` only if it stays narrowly focused on the missing capability: isolation-aware routing plus OpenCode session/workspace fan-out and fan-in. Treat `opencode-forge` as the primary implementation reference for workspace/session mechanics and consider contributing this as a Forge mode before creating a new standalone package.

## Source-Backed OpenCode Capabilities

| Capability | Source Of Truth | Notes |
| --- | --- | --- |
| Experimental workspace endpoints | `packages/opencode/src/server/routes/instance/httpapi/groups/workspace.ts` | Routes live under `/experimental/workspace`: adapter list, list, create, sync-list, status, remove, warp. |
| Workspace HTTP handlers | `packages/opencode/src/server/routes/instance/httpapi/handlers/workspace.ts` | `create` calls `Workspace.Service.create`; `warp` binds a session to a workspace. |
| Workspace adapter type | `packages/opencode/src/control-plane/types.ts` | Adapter has `configure`, `create`, optional `list`, `remove`, `target`. |
| Built-in worktree adapter | `packages/opencode/src/control-plane/adapters/worktree.ts` | Adapter type is `worktree`; target is local directory. |
| Worktree creation internals | `packages/opencode/src/worktree/index.ts` | Creates worktree under OpenCode data dir, can preserve `branch`, emits `worktree.ready` / `worktree.failed`. |
| Workspace plugin registration | `packages/opencode/src/plugin/index.ts` | Server plugin input exposes `experimental_workspace.register(type, adapter)`. |
| Runtime feature flag | `packages/opencode/src/effect/runtime-flags.ts` | `OPENCODE_EXPERIMENTAL_WORKSPACES` gates workspace sync/runtime behavior. |
| Background subagent flag | `packages/opencode/src/effect/runtime-flags.ts`, `packages/opencode/src/tool/task.ts` | `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS` is required when the built-in `task` tool uses `background=true`. |
| V2 SDK workspace API | `packages/sdk/js/src/v2/gen/sdk.gen.ts` | `experimental.workspace.create/list/status/remove/warp/syncList`. |
| V2 SDK session create | `packages/sdk/js/src/v2/gen/sdk.gen.ts` | `session.create` accepts `parentID`, `title`, `agent`, `model`, `metadata`, `permission`, `workspaceID`. |
| V2 SDK async prompt | `packages/sdk/js/src/v2/gen/sdk.gen.ts` | `session.promptAsync` queues a message and returns immediately. |
| Server plugin client caveat | `packages/opencode/src/plugin/index.ts` | Plugin input currently creates `client` from `@opencode-ai/sdk`; do not assume it exposes V2 experimental workspace APIs. Use `@opencode-ai/sdk/v2` or raw HTTP when needed. |
| Background subagents | `packages/opencode/src/tool/task.ts` | Useful for `same-session-subagent` UX: the tool creates a child task/session under the hood and injects the result back into the parent session; it does not create isolated worktree workspaces. |
| Session identity fields | `packages/opencode/src/session/session.ts` | Session records include `title`, `parentID`, `metadata`, and `workspaceID`; use these to mark orchestrator-created workers. |
| App root session list | `packages/app/src/pages/layout/helpers.ts` | Shared app UI filters root-visible sessions with `!session.parentID`; child sessions may not appear as normal root sessions in the sidebar. |
| Workspace/session events | `OpenCode/03-events-reference.md` | Prefer observable `workspace.status`, `worktree.ready/failed`, `session.created/updated/status`, `message.*`. `workspace.ready/failed` types exist in source, but do not rely on them unless emission is verified for the target OpenCode version. |
| Desktop vs TUI limitation | `OpenCode/10-desktop-development.md` | TUI plugins do not render in Desktop/Web; server plugin behavior can still affect Desktop sessions. |

## Non-Goals

| Non-Goal | Reason |
| --- | --- |
| Driving UI clicks | Use SDK/HTTP APIs instead of simulating `New workspace` button clicks. |
| Replacing `opencode-swarm` QA framework | `orchestrator` should coordinate isolation-aware session/workspace fan-out, not replicate all guardrails. |
| Creating a worktree for every worker | Worktree/workspace isolation is an escalation mode, not the default for every small subtask. |
| Replacing `opencode-forge` loop engine | Reuse or borrow its workspace/session loop patterns. |
| Automatic merge without review | Parallel workers can create conflicting diffs; master must require review/merge readiness. |
| Desktop visual plugin on day one | No public Desktop/Web plugin surface exists. Use server tools and optional TUI plugin first. |
| Remote/cloud workers in v1 | Start with local git worktrees. Remote adapters can come later. |
| Exposing raw OpenCode SDK as the model-facing API | Models should call stable `orchestrator_*` tools, not low-level `session.create`, `workspace.create`, or `promptAsync` primitives. |

## Product Shape

Final product should be a hybrid: server plugin for runtime execution plus a skill for strategy/instructions.

| Artifact | Required? | Role |
| --- | --- | --- |
| Server plugin | Yes | Runtime executor: tools, state, events, permissions, child sessions, workspace escalation, report validation, status/collect/cancel/cleanup. |
| Bundled skill | Yes | Strategy layer: when to use `task`, when to call `orchestrator_start`, how to choose isolation mode, how to write worker prompts, and how to interpret reports. |
| TUI plugin | Optional | Command palette, dialogs, worker status panel, jump-to-worker actions. |
| Desktop/Web UI | No for v1 | Desktop users can still invoke server tools from chat; no custom visual surface. |
| External CLI | Optional later | Batch launch, CI integration, recovery/cleanup scripts. |

Recommended repository/package layout:

```text
opencode-orchestrator/
  plugin/
    server.ts
    tui.ts
  skills/
    orchestrator/SKILL.md
  README.md
```

Installation should make both layers available. The plugin may be configured in `opencode.json`; the skill can be bundled through `skills.paths`, installed globally, or documented as a project-local skill depending on packaging constraints.

## Architecture

`orchestrator` should be a multi-surface package:

| Surface | Required? | Role |
| --- | --- | --- |
| Server plugin | Yes | Core orchestration, tools, state, events, workspace/session API calls, safety checks. |
| Skill | Yes | Model-facing operating manual for the `orchestrator_*` tools and isolation ladder. |
| TUI plugin | Optional | Command palette, dialogs, worker status panel, jump-to-worker actions. |
| Desktop/Web UI | No for v1 | Desktop users can still invoke server tools from chat; no custom visual surface. |
| External CLI | Optional later | Batch launch, CI integration, recovery/cleanup scripts. |

Package shape should follow OpenCode plugin constraints: split server and TUI entrypoints; do not put both `server` and `tui` in one default plugin object.

## Model-Facing Contract

The model should learn `orchestrator` from two sources:

| Source | What The Model Sees | Purpose |
| --- | --- | --- |
| Plugin tool definitions | Tool names, descriptions, and argument schemas for `orchestrator_*`. | Gives the model safe executable actions. |
| Skill instructions | Decision ladder, examples, worker prompt templates, report contract, and when not to use `orchestrator`. | Teaches the model when and how to use those actions. |

The model-facing API should be intentionally high-level. The model should not be asked to call OpenCode SDK primitives directly during normal operation.

Good model-facing flow:

```text
Model reads orchestrator skill -> chooses isolation mode -> calls orchestrator_start(plan) -> plugin executes SDK calls and stores state.
```

Bad model-facing flow:

```text
Model reads raw SDK signatures -> writes ad-hoc node script -> manually calls session.create/workspace.create/promptAsync/polling.
```

## Plugin Responsibilities

The plugin is the runtime executor. It owns behavior that cannot be made reliable with prose instructions alone.

| Responsibility | Why It Belongs In Plugin |
| --- | --- |
| `orchestrator_start` | Validates the approved plan, creates run state, creates child sessions/workspaces as needed, and dispatches prompts. |
| `orchestrator_status` | Reads durable run state plus live events/polling to report worker status. |
| `orchestrator_collect` | Parses report envelopes, validates worker identity, gathers diffs/tests, and produces a master summary. |
| `orchestrator_cancel` | Cancels selected worker sessions/background jobs and updates run state. |
| `orchestrator_cleanup` | Removes only approved completed workspaces/worktrees and optional audit exports. |
| State | Persists run/worker/session/workspace IDs across compaction and OpenCode restart. |
| Event handling | Tracks `session.status`, `message.*`, `permission.*`, `question.*`, workspace/worktree events, with polling fallback. |
| Enforcement | Blocks recursive starts, validates caller role, checks scopes, and keeps cleanup/merge authority on the master side. |

## Skill Responsibilities

The skill is the operating manual. It should be short enough to load when relevant, and should avoid copying low-level SDK implementation details.

| Responsibility | What The Skill Should Teach |
| --- | --- |
| Trigger conditions | Use `orchestrator` for multi-worker tasks that need coordination, status, report collection, or isolation escalation. |
| Cheap path | Prefer built-in `task`/subagents for small read-only or research work when no durable orchestration is needed. |
| Isolation ladder | Choose `same-session-subagent`, then `same-workspace-session`, then workspace mode only when risk/context/diff isolation requires it. |
| Plan format | Ask the model to propose workers with scope, mode, dependencies, risk, verification, and report expectations. |
| Tool usage | Show examples of `orchestrator_start`, `orchestrator_status`, and `orchestrator_collect` with valid argument shapes. |
| Worker prompt contract | Require bounded mission, allowed/forbidden files, constraints, verification, and one final report envelope. |
| Stop conditions | Ask the user when scopes overlap, workspace flags are missing, or the task is too ambiguous to parallelize safely. |

## Why Not Skill-Only

A skill is Markdown context, not a runtime service. It can guide the model, but it cannot reliably provide orchestration mechanics.

| Missing Capability | Why Skill Alone Is Not Enough |
| --- | --- |
| Durable state | A skill cannot persist authoritative run/worker/session/workspace state across restart or compaction. |
| Event subscription | A skill cannot subscribe to OpenCode events and continuously update worker state. |
| Safe tool contract | A skill cannot expose typed `orchestrator_start/status/collect` schemas; it can only describe what the model should do. |
| Enforcement | A skill cannot hard-block recursive starts, unsafe cleanup, or out-of-scope writes. |
| Reliable completion | A skill cannot independently validate report envelopes, session IDs, and workspace IDs. |
| Recovery | A skill cannot reconcile live sessions/workspaces after OpenCode restarts. |

Skill-only is acceptable for a spike or for simple same-session `task` usage. It is not enough for a product that promises child sessions, workspace escalation, status, collect, cancel, cleanup, and recovery.

## Why Not Raw SDK From Skill

The skill could instruct the model to run a Node script that calls OpenCode SDK methods. That can be useful for a spike, but it should not be the final model-facing interface.

| Problem | Consequence |
| --- | --- |
| Ad-hoc scripts vary by run | No stable schema for the model, no consistent status/collect behavior. |
| Model owns plumbing | The model must remember SDK call order, routing, polling, parsing, permissions, and cleanup every time. |
| Weak safety | Permission and role checks become optional script logic instead of centralized plugin enforcement. |
| Poor lifecycle | A one-shot script may dispatch workers and exit, leaving no resident owner for state/events/recovery. |
| Context bloat | Raw SDK signatures in skill context distract the model from planning and coordination. |

If a Node CLI grows enough to manage state, events, status, and cleanup, it has effectively become an external orchestrator. For OpenCode-native UX, a server plugin is the better home because it runs inside the tool/session/permission/event lifecycle.

## Control Plane

### Master Session

The master session is the only place where the user gives the global goal and approves fan-out. It owns:

| Responsibility | Detail |
| --- | --- |
| Decomposition | Turn user goal into worker tasks with file scopes and dependencies. |
| Admission control | Enforce max workers, max cost/time, allowed paths, and risk classes. |
| Isolation routing | Pick the least expensive execution mode that keeps context, diffs, and risk manageable. |
| Session/workspace creation | Create only the child sessions and worktree workspaces required by the approved plan. |
| Report aggregation | Collect reports and produce a single master status/merge plan. |
| Human escalation | Ask questions when workers conflict, block, or need permission. |

### Worker Session

Each worker receives one bounded task and an execution mode. A worker can be a same-session subagent UX task, a child session in the current workspace, a child session in a new worktree workspace, or one of several child sessions inside a new worktree workspace.

Worker prompt contract:

| Required Section | Purpose |
| --- | --- |
| Mission | One clear task, no side quests. |
| Scope | Allowed files/directories and explicitly forbidden files. |
| Dependencies | Upstream worker outputs that must be respected. |
| Constraints | No push, no merge to main, no modifying `.orchestrator/`; submit reports through the assigned worker report channel. |
| Verification | Commands/tests expected for this worker. |
| Report format | Structured final report required before stopping. |

Final report contract:

| Field | Required | Meaning |
| --- | --- | --- |
| `status` | Yes | `done`, `blocked`, `failed`, `needs-review`. |
| `runID` | Yes | Orchestrator run ID. |
| `workerID` | Yes | Stable worker ID, such as `w01`. |
| `mode` | Yes | `same-session-subagent`, `same-workspace-session`, `new-workspace-session`, or `new-workspace-multi-session`. |
| `summary` | Yes | Short human-readable result. |
| `sessionID` | Yes | Worker session ID. For same-session subagent UX, use the child task/session ID created by the `task` behavior. If no concrete worker session/task ID is available, do not run that worker in parallel. |
| `workspaceID` | Workspace modes | OpenCode workspace ID when a new workspace is created. Same-workspace workers can record the current workspace ID if available. |
| `worktree` | Workspace modes | Local worktree path when a worktree workspace exists. |
| `branch` | Branch-backed modes | Worker branch if branch-backed. |
| `baseRevision` | Mutating workers | Git revision used as the worker base. |
| `headRevision` | Mutating workers | Worker branch HEAD if committed, or `uncommitted` if work remains only in the current worktree. |
| `filesChanged` | Yes | Relative paths changed, or empty list for read-only/research workers. |
| `diffRef` | Mutating workers | Path or command for reviewing the worker diff/patch. |
| `testsRun` | Yes | Commands and pass/fail result, or `not run` with reason. |
| `blockers` | Yes | Anything master/user must resolve. |
| `mergeNotes` | Yes | How to integrate safely, or `none` for read-only workers. |

Report envelope rule:

```text
<ORCHESTRATOR_REPORT version="1">
{
  "runID": "orch-20260606-120000-auth",
  "workerID": "w01",
  "mode": "same-workspace-session",
  "sessionID": "ses_worker",
  "status": "done",
  "summary": "Updated auth tests",
  "filesChanged": ["src/auth.test.ts"],
  "testsRun": ["npm test -- auth: pass"],
  "blockers": [],
  "mergeNotes": "Review test diff before merge"
}
</ORCHESTRATOR_REPORT>
```

The worker must emit exactly one final report envelope. The JSON body must satisfy the report contract above. The master must ignore partial streaming text, parse only a complete envelope, and reject reports whose `runID`, `workerID`, `mode`, `sessionID`, or `workspaceID` do not match durable run state.

## Execution Mode Ladder

`orchestrator` should escalate isolation only when the cheaper mode is not enough.

| Mode | Use When | Mechanism | Creates Worktree? |
| --- | --- | --- | --- |
| `same-session-subagent` | Small/read-only/research tasks, or scoped edits that do not need separate chat history from the user's point of view. | Prefer OpenCode background subagent behavior through the built-in `task` tool semantics: the worker runs as a child task/session under the hood and injects the result back into the master session. | No |
| `same-workspace-session` | Contexts would pollute each other, but all workers can safely share the same branch/workspace. | Create a child session with `parentID`, title, metadata, permissions, and no new workspace. Dispatch with `session.promptAsync`. | No |
| `new-workspace-session` | The task needs branch/diff isolation, rollback safety, risky mutation, or conflicts with current work. | Create a worktree workspace, wait for readiness, create one child session in it, then dispatch with `session.promptAsync`. | Yes |
| `new-workspace-multi-session` | A large isolated branch contains subpackages that can themselves be parallelized. | Create one worktree workspace, then run one or more child sessions inside it using the same ladder recursively, but do not spawn another workspace unless explicitly needed. | Yes, one for the group |

Decision tree:

1. If the worker is read-only, research-only, or can return a compact answer to the master, use `same-session-subagent`.
2. If the worker needs its own long context but can share the current branch and filesystem state, use `same-workspace-session`.
3. If the worker needs branch isolation, changes global files, may break local state, or should be reviewed as a separate diff, use `new-workspace-session`.
4. If the isolated branch is itself large enough to split, use `new-workspace-multi-session` inside that workspace.
5. If scopes overlap on the same mutating files, do not run those workers in parallel; serialize or ask the user.

Hard requirement for parallel same-session subagent mode: background `task` workers require `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true`. Without it, `same-session-subagent` can still be used as a foreground/serial helper, but it is not a parallel fan-out mode; choose `same-workspace-session` or ask the user.

### E2E Orchestration

| Step | Behavior |
| --- | --- |
| Plan | Master analyzes the task and proposes workers with scope, risk, dependencies, and execution mode. |
| Approve | User approves the plan or edits modes/scopes/max parallelism. |
| Record state | Plugin creates a run in its durable state before dispatching workers. |
| Dispatch cheap modes | `same-session-subagent` workers are launched first when they can unblock planning or discovery. |
| Dispatch session modes | `same-workspace-session` workers are created with `session.create({ parentID, title, metadata, permission })`, then prompted asynchronously. |
| Dispatch workspace modes | Workspace workers first go through workspace creation/readiness, then session creation and prompt dispatch. |
| Monitor | Events update worker status; polling reconciles missed events and restart recovery. |
| Complete | Master collects structured reports, validates required fields, reads diffs/tests, and produces the merge plan. |

### Workspace Strategy For Escalated Isolation

Use the built-in OpenCode worktree adapter only for modes that need branch/worktree isolation:

| Step | API/Behavior |
| --- | --- |
| Verify support | `experimental.workspace.adapter.list` should include `worktree`. |
| Create workspace | `experimental.workspace.create` with type `worktree`, branch `orchestrator/<run>/<worker-or-group>` or detached fallback. |
| Wait for readiness | For local `worktree`, do not treat `workspace.status=connected` as sufficient. Use it as a connection signal, then wait for `worktree.ready` or verified worktree bootstrap completion before dispatching prompts. |
| Create child session | `session.create` with body `workspaceID`, `parentID`, title, metadata, and worker permission rules; route the request with workspace/directory context when using raw HTTP/SDK parameters. |
| Send task | `session.promptAsync` with `sessionID`, workspace/directory context, and worker prompt parts. |
| Keep synchronized | `experimental.workspace.syncList` after creation and after major lifecycle events so workspace APIs/status stay current. Do not assume this alone makes the workspace appear in every Desktop sidebar/list. |

Hard requirement for workspace modes: launch OpenCode with `OPENCODE_EXPERIMENTAL_WORKSPACES=true` or `OPENCODE_EXPERIMENTAL=true` for workspace switching/sync. Without it, the plugin should fail clearly before choosing workspace modes, or degrade to non-workspace modes only if the user explicitly approves the risk.

### Branch Naming

Use deterministic branch/worktree names:

| Entity | Pattern |
| --- | --- |
| Run ID | `orch-YYYYMMDD-HHMMSS-shortgoal` |
| Worker ID | `w01`, `w02`, `w03` |
| Branch | `orchestrator/<run-id>/<worker-id>` |
| Session title | `[orchestrator <run-id> <worker-id>] <short task>` |

Do not rely on model-generated names for file paths or branch names. Slug and length-limit every value.

## Plugin Tools

The model-facing API should be small. Avoid exposing low-level primitives like raw workspace creation to arbitrary agents.

| Tool | Who Calls It | Purpose |
| --- | --- | --- |
| `orchestrator_start` | Master session only | Validate plan, create run, create workspaces/sessions, dispatch prompts. |
| `orchestrator_status` | Master or user | Show run and worker states. |
| `orchestrator_collect` | Master session | Aggregate final reports and diffs. |
| `orchestrator_cancel` | User/master | Stop selected workers and mark run cancelled. |
| `orchestrator_cleanup` | User/master | Remove completed workspaces/worktrees after merge or discard. |
| `orchestrator_merge_plan` | Master session | Produce safe integration order and conflict list; does not merge automatically in v1. |

Tool permission policy:

| Rule | Reason |
| --- | --- |
| Only master session can call lifecycle tools by default. | Prevent recursive swarms. |
| Worker sessions cannot call `orchestrator_start`. | Avoid runaway nested workspace creation. |
| Worker sessions cannot remove workspaces. | Cleanup is coordinator/user-owned. |
| Worker sessions get scoped edit permissions. | Reduces parallel conflict blast radius. |
| Dangerous shell commands stay denied or ask. | Worktree isolation is not a security boundary. |

Enforcement requirements:

| Check | Where To Enforce |
| --- | --- |
| Caller role | `orchestrator_*` tool handlers must verify `ctx.sessionID` or input `sessionID` against durable run state before doing lifecycle work. Do not rely on prompt instructions. |
| Worker identity | Store `sessionID`, `workspaceID`, `parentID`, worker ID, and role in run metadata; reject calls when they do not match the current run. |
| Scoped writes | Enforce with permission config and/or `tool.execute.before` path checks, not only worker prompt wording. |
| Cleanup/merge authority | Require master/user-owned tool calls and completed/blocked run state before removing workspaces or producing integration instructions. |

## State Model

State must survive OpenCode restart and worker session compaction.

Preferred storage:

| Store | Location | Notes |
| --- | --- | --- |
| Runtime DB | OpenCode/plugin data dir | Authoritative durable run/worker/session/workspace state; not intended for git. |
| Project audit export | `.orchestrator/` in the master workspace | Optional human-readable export for reports and merge plans; auto-add to `.git/info/exclude` or ask before committing. |
| Worker reports | Runtime DB first; optional `.orchestrator/runs/<run-id>/reports/<worker-id>.md` export | Avoid treating each worker worktree as an independent source of truth. Master collection owns the export. |
| Event log | Runtime DB first; optional `.orchestrator/runs/<run-id>/events.jsonl` export | Append-only, useful for recovery/debug, but plugin state remains authoritative. |

Minimum run state:

| Field | Meaning |
| --- | --- |
| `runID` | Stable run identifier. |
| `masterSessionID` | Coordinator session. |
| `goal` | Original user goal. |
| `workers[]` | Worker IDs with task, mode, scope, sessionID, optional workspaceID/branch, base/head revisions, diffRef, status, and last observed event/message. |
| `createdAt/updatedAt` | Recovery and cleanup. |
| `maxParallel` | Admission limit used. |
| `status` | `planning`, `running`, `collecting`, `done`, `cancelled`, `failed`. |

## Completion Signaling

The master session must not rely on a single event to know that a worker finished. V1 should combine structured reports, events, polling, and durable state.

| Signal | Use | Caveat |
| --- | --- | --- |
| Structured final report | Authoritative worker completion payload. Master parses the complete `<ORCHESTRATOR_REPORT version="1">` envelope and validates `runID`, `workerID`, `mode`, `sessionID`, workspace fields, status, files, tests, blockers, and merge notes. | The report can be malformed or missing; then worker remains `needs-review` or `blocked`. |
| `session.status` | Detect child session `busy`, `idle`, or `retry`. | `idle` means no active turn, not necessarily successful completion. Master still checks report. |
| `message.updated` / `message.part.updated` | Detect report markers while the assistant message streams or updates. | Streaming can be partial; validate only once the worker is idle or the report is complete. |
| Built-in task result injection | For `same-session-subagent`, the `task` behavior injects a synthetic result into the parent session, including the child task/session ID. Parse the report envelope from that injected result. | Treat this as a report transport, not as full run state. |
| `permission.asked` / `question.asked` | Mark worker blocked and surface the blocker to master/user. | A pending permission/question is not failure. |
| `session.error` | Mark worker failed and capture error details. | Recovery may still be possible from durable state. |
| Polling fallback | Re-read child sessions/messages/workspaces after missed events or restart. | Required because plugin/server can restart and event schemas can drift. |

Worker completion algorithm:

1. Dispatch worker and store run state before sending the prompt.
2. Mark worker `running` when prompt dispatch succeeds.
3. Watch events and periodically poll worker session/message state.
4. When a worker becomes idle or a task result arrives, parse exactly one complete `<ORCHESTRATOR_REPORT version="1">...</ORCHESTRATOR_REPORT>` envelope from the final worker output.
5. Validate the report against the stored `runID`, `workerID`, `mode`, `sessionID`, and `workspaceID` if present; reject mismatches as `needs-review`.
6. Mark worker `done`, `blocked`, `failed`, or `needs-review` in durable state.
7. Notify the master session through `orchestrator_status`/`orchestrator_collect` output, and optionally inject a concise synthetic status update if the implementation supports it safely.

## Scheduling And Conflict Policy

V1 should be conservative.

| Rule | Default |
| --- | --- |
| Max workers | 2-4 unless user explicitly raises it. |
| Require file scopes | Yes. No scope means serial or ask user. |
| Read-only research | Prefer `same-session-subagent`. |
| Context-heavy but same branch | Prefer `same-workspace-session`. |
| Global files | Serialize tasks touching package managers, lockfiles, root config, auth/security; if parallelism is still needed, escalate to workspace mode. |
| Same file overlap | Do not run in parallel. |
| Parent/child directory overlap | Do not run in parallel unless read-only. |
| Dependencies | Use DAG waves; only dispatch next wave after dependencies report done. |
| Dirty main worktree | Ask before mutating same-workspace workers, or require clean base for workspace modes. |

Borrow concepts from `opencode-swarm` Lean Turbo lane planning, but implement only what is needed for isolation-aware session/workspace fan-out.

## Event Handling

The server plugin should subscribe to event bus updates and maintain derived worker state.

| Event | Use |
| --- | --- |
| `workspace.status` | Track `connected`, `connecting`, `disconnected`, `error`; for worktrees this is not enough to prove checkout/bootstrap readiness. |
| `worktree.ready` / `worktree.failed` | Worktree-specific diagnostics. |
| `session.created` | Confirm child session registration. |
| `session.status` | Detect worker busy/idle/stalled. |
| `message.updated` / `message.part.updated` | Detect final report markers and stream status. |
| `permission.asked` / `question.asked` | Surface worker blockers to master/user. |
| `session.error` | Mark worker failed and capture reason. |

Use polling as fallback because event payloads can drift across OpenCode versions. `workspace.ready` / `workspace.failed` event types exist in source, but do not make them the primary readiness signal until emission is verified in the target version.

## UX Surfaces

### Chat/Tool UX

Works in TUI, CLI, Desktop, and Web because it is server-side.

| User Intent | Expected Behavior |
| --- | --- |
| “Run this with orchestrator” | Master proposes worker plan and asks for approval. |
| “Status” | Master calls `orchestrator_status` and summarizes workers. |
| “Collect results” | Master calls `orchestrator_collect`, reads reports/diffs, produces merge plan. |
| “Cancel worker 2” | Master calls `orchestrator_cancel` for that worker. |

### TUI Plugin UX

Optional but valuable:

| TUI Feature | Purpose |
| --- | --- |
| Command palette `Orchestrator: Start` | Opens a prompt/dialog for goal and max workers. |
| Sidebar/status slot | Shows worker count, running/done/blocked. |
| Worker list dialog | Jump to worker session/workspace. |
| Toasts | Workspace creation, worker done/blocked/failure. |

Use current TUI keymap API (`api.keymap.registerLayer`). TUI code should not assume OpenCode plugin tools are a general TUI-to-server RPC surface. For v1, prefer prompt-mediated UX: the TUI collects input and sends a master-session prompt that causes the model to call `orchestrator_start`. For zero-LLM direct start, the package must provide an explicit supported bridge, such as shared core code with a server-owned endpoint/action. TUI may read status through supported SDK/state paths, but it must not duplicate workspace/session creation logic or bypass server-side safety checks. Do not use deprecated `api.command.*`.

### Desktop/Web UX

No custom Desktop/Web visual plugin for v1. Desktop sessions can still use server tools and worker sessions because Desktop talks to the same OpenCode server/sidecar. If Desktop-specific UI is required later, it needs a core `packages/app` / `packages/desktop` change or future Desktop extension point.

### Desktop Visibility And Identification

Orchestrator-created sessions should be visible through OpenCode's normal server/app state when they are created in the same Desktop sidecar/server. Workspace-mode work should be available through workspace APIs/status and `orchestrator_status`; do not promise that server-created workspaces automatically appear in every current Desktop sidebar/list without a verified app integration. Do not promise a custom Desktop panel in v1.

Important nuance: child sessions have `parentID`. The shared app's root session list filters for root sessions, so child sessions may appear as related/child task sessions, notifications, direct session links, or through `orchestrator_status`, but not necessarily as ordinary user-created root sessions in the sidebar.

User-facing identification requirements:

| Surface | Requirement |
| --- | --- |
| Session title | Prefix every worker title with `[orchestrator <runID> <workerID>]`, for example `[orchestrator orch-20260606-auth w02] update auth tests`. This is the fallback when metadata is not shown by UI. |
| Session metadata | Store `orchestrator: true`, `runID`, `workerID`, `role: "worker"`, `mode`, `masterSessionID`, optional `workspaceID`, and optional `branch`. |
| Parent linkage | Set `parentID` to the master session for worker child sessions whenever possible. |
| Workspace/branch names | Use `orchestrator/<run-id>/<worker-id>` or `orchestrator/<run-id>/<group-id>` for workspace-mode work. |
| Master output | `orchestrator_start` and `orchestrator_status` must list every worker with session ID, title, mode, status, workspace/branch when present, and a copyable/openable session reference if the client supports it. |
| Audit exports | Optional `.orchestrator/` exports should use the same run/worker IDs so Desktop-visible names and filesystem artifacts match. |

Never rely on the user manually distinguishing worker sessions by memory. If the UI only shows session titles, the title alone must be enough to tell that a session was created by orchestrator rather than manually by the user.

## Failure Modes

| Failure | Handling |
| --- | --- |
| Experimental workspace flag missing | Do not choose workspace modes. Continue only with approved non-workspace modes, or ask the user to enable the flag. |
| Background subagent flag missing | Do not claim same-session parallel fan-out. Use foreground/serial task behavior, choose `same-workspace-session`, or ask user to enable `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true`. |
| Adapter unavailable | Tell user `worktree` adapter is not listed for workspace-mode workers; do not manually shell out unless configured. |
| Mode misclassification | Stop before dispatch if scope/risk says workspace isolation is needed but unavailable or unapproved. |
| Workspace create fails | Mark the workspace-mode worker failed, keep run active if other workers can proceed. |
| Workspace/worktree readiness timeout | Retry status/readiness polling, then mark blocked; do not send prompt blindly. |
| Child session create fails | For workspace modes, remove workspace if empty; mark worker failed. For same-workspace modes, keep master state inspectable. |
| Prompt dispatch fails | Keep workspace/session for inspection; mark failed. |
| Report missing or malformed | Mark worker `needs-review`; do not treat `idle` as success. |
| Permission/question pending in worker | Pause worker and surface blocker to master/user. |
| Worker stalls | Detect idle/no progress timeout; allow restart or cancel. |
| Conflicting diffs | Produce merge plan; do not auto-merge. |
| OpenCode restart | Recover from durable state and re-list workspaces/sessions. |

## Security And Safety

| Concern | Requirement |
| --- | --- |
| Worktree isolation | Treat as conflict isolation, not a sandbox. Same user, same filesystem. |
| Secrets | Do not copy ignored secrets automatically in v1; document optional integration with devcontainer/worktree env setup. |
| Network and push | Deny `git push` and production deploys in worker sessions by default. |
| State writes | Workers cannot modify `.orchestrator/`; master/plugin collection owns audit exports. |
| Recursive spawning | Workers cannot call `orchestrator_start`; master-only lifecycle tools. |
| External directories | Deny or ask; worktree path should be the trusted root. |
| Cost runaway | Max workers, max runtime, max tool calls, max retries. |

## Open Questions For Product Design

These are not blockers for the design doc, but should be decided before implementation:

1. Should `orchestrator` be a standalone package or an `opencode-forge` contribution/mode?
2. What exact thresholds should choose `same-session-subagent`, `same-workspace-session`, or workspace mode?
3. For workspace-mode workers, should worktrees be branch-backed by default, or detached until merge time?
4. Should worker prompts be generated by the master model or deterministic from a user-approved plan file?
5. Should v1 support Desktop users only through chat tools, or also require TUI for the full workflow?
6. Should `.orchestrator/` audit exports be generated automatically, only on demand, or disabled by default?
7. Should `orchestrator` integrate with existing `opencode-swarm`/`swarm-tools` file reservation formats, or keep its own minimal scope model?

## MVP Plan

| Phase | Deliverable | Exit Criteria |
| --- | --- | --- |
| 0. Spike | Prove non-workspace fan-out. | One master action dispatches two workers without creating worktrees: one same-session subagent UX worker and one same-workspace child session. |
| 1. Server MVP | Server plugin tools, durable state, and completion signaling. | `orchestrator_start/status/collect/cancel` work from chat and correctly parse worker reports. |
| 2. Isolation Routing MVP | Mode selection and scope validation. | Planner chooses the cheapest safe mode and refuses unsafe parallel overlap. |
| 3. Workspace Escalation MVP | Worktree workspace mode. | A risky worker creates one worktree workspace, one child session, and reports back. |
| 4. Safety MVP | Permission rules and hard enforcement. | Workers cannot edit outside assigned scope or start nested orchestrator runs. |
| 5. TUI MVP | Command palette/dialog/status/jump. | User can launch, monitor, and jump to worker sessions/workspaces from TUI. |
| 6. Recovery | Restart/reconcile. | Plugin can recover active runs after OpenCode restart. |
| 7. Merge Assist | Aggregated diff and merge plan. | Master produces deterministic integration order and conflict report. |

## Verification Checklist

Use a test repo with a clean git status for mutating tests. Read-only mode tests can run without workspace flags.

1. For parallel same-session tests, start OpenCode with `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true`.
2. Start a run with two read-only/research workers and confirm no worktree workspace is created.
3. Confirm `same-session-subagent` workers have concrete child task/session IDs and return a complete `<ORCHESTRATOR_REPORT version="1">` envelope to the master session.
4. Start a run with two context-heavy but same-branch workers and confirm child sessions are created with `parentID` and no new workspace.
5. Confirm each child session receives its prompt through `promptAsync` and begins running.
6. Confirm `orchestrator_status` reports running/done/blocked accurately across both non-workspace modes.
7. Start OpenCode with `OPENCODE_EXPERIMENTAL_WORKSPACES=true` for workspace-mode tests.
8. Confirm `experimental.workspace.adapter.list` includes `worktree`.
9. Start a risky mutating worker and confirm exactly one worktree workspace is created for that worker or worker group.
10. Confirm the workspace reaches `connected`, and for worktrees completes `worktree.ready` or equivalent bootstrap verification before prompt dispatch.
11. Confirm the workspace-mode child session is created with `workspaceID`, `parentID`, and correct workspace/directory routing.
12. Force one worker failure and confirm the run remains inspectable.
13. Force one permission/question blocker and confirm it surfaces to master/user.
14. Confirm missing, malformed, partial, or mismatched reports become `needs-review`, not `done`.
15. Confirm `orchestrator_collect` reads reports and produces a merge plan.
16. Confirm `orchestrator_cleanup` removes only selected completed workspaces, not same-workspace sessions.
17. Restart OpenCode mid-run and confirm recovery/reconciliation.

Relevant upstream tests to study or mirror in the future project:

| Test Area | Upstream Test Names From Source Search |
| --- | --- |
| Workspace adapter | `packages/opencode/test/plugin/workspace-adapter.test.ts` |
| HTTP workspace API | `packages/opencode/test/server/httpapi-workspace.test.ts` |
| Workspace routing | `packages/opencode/test/server/workspace-routing.test.ts` |
| TUI workspace create | `packages/opencode/test/cli/cmd/tui/dialog-workspace-create.test.ts` |

## Decision

Build `orchestrator` only if the exact isolation-aware fan-out/fan-in workflow is required and cannot be contributed to `opencode-forge`. The project should be intentionally thin: use OpenCode's existing subagent/session/workspace APIs, expose a small `orchestrator_*` tool surface, bundle a strategy skill, reuse proven patterns from Forge and Swarm, and focus on master-session orchestration plus report aggregation.
