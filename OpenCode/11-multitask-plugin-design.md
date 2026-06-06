# Multitask Plugin Design

Design-only brief for a future OpenCode plugin named `multitask`.

Goal: one master OpenCode session decomposes a large task, creates multiple isolated worker workspaces backed by git worktrees, creates one OpenCode session in each workspace, dispatches work in parallel, tracks status, and aggregates worker reports back into the master session.

No implementation code belongs in this document. Use it as the source pack for a new repository.

Source note: OpenCode anchors were checked against local upstream checkout `18ba80f`; external package review was last checked on 2026-06-06 and can drift faster than OpenCode source APIs.

## Task Clarity

The task is clear.

Desired user-facing behavior:

1. User starts in a master session and asks for a large task to be parallelized.
2. `multitask` plans independent work packages with file scopes, dependencies, models/agents, and risk labels.
3. After user approval, `multitask` creates one OpenCode workspace per worker, usually via git worktree.
4. Inside each workspace, `multitask` creates a new OpenCode session and sends a worker prompt asynchronously.
5. The master session remains the coordinator: it shows status, handles blockers, collects reports, and decides when to merge or ask the user.
6. Each worker session ends with a structured report: summary, changed files, tests run, blockers, branch/worktree, merge readiness.

Important translation: the plugin should not literally click the UI button `New workspace`. It should use the same underlying OpenCode workspace/session APIs. UI automation is fragile; API calls are testable and work from server/TUI code.

## Existing Solutions Review

Exact match found: no.

Close matches worth reusing or studying:

| Package | What Exists | Useful For `multitask` | Gap |
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

Recommendation: do not build a large swarm framework from scratch. Build `multitask` only if it stays narrowly focused on the missing capability: OpenCode workspace/session fan-out and fan-in. Treat `opencode-forge` as the primary implementation reference and consider contributing this as a Forge mode before creating a new standalone package.

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
| V2 SDK workspace API | `packages/sdk/js/src/v2/gen/sdk.gen.ts` | `experimental.workspace.create/list/status/remove/warp/syncList`. |
| V2 SDK session create | `packages/sdk/js/src/v2/gen/sdk.gen.ts` | `session.create` accepts `parentID`, `title`, `agent`, `model`, `metadata`, `permission`, `workspaceID`. |
| V2 SDK async prompt | `packages/sdk/js/src/v2/gen/sdk.gen.ts` | `session.promptAsync` queues a message and returns immediately. |
| Server plugin client caveat | `packages/opencode/src/plugin/index.ts` | Plugin input currently creates `client` from `@opencode-ai/sdk`; do not assume it exposes V2 experimental workspace APIs. Use `@opencode-ai/sdk/v2` or raw HTTP when needed. |
| Background subagents | `packages/opencode/src/tool/task.ts` | Useful comparison point, but built-in `task` creates child sessions, not isolated worktree workspaces. |
| Workspace/session events | `OpenCode/03-events-reference.md` | Prefer observable `workspace.status`, `worktree.ready/failed`, `session.created/updated/status`, `message.*`. `workspace.ready/failed` types exist in source, but do not rely on them unless emission is verified for the target OpenCode version. |
| Desktop vs TUI limitation | `OpenCode/10-desktop-development.md` | TUI plugins do not render in Desktop/Web; server plugin behavior can still affect Desktop sessions. |

## Non-Goals

| Non-Goal | Reason |
| --- | --- |
| Driving UI clicks | Use SDK/HTTP APIs instead of simulating `New workspace` button clicks. |
| Replacing `opencode-swarm` QA framework | `multitask` should orchestrate workspace/session fan-out, not replicate all guardrails. |
| Replacing `opencode-forge` loop engine | Reuse or borrow its workspace/session loop patterns. |
| Automatic merge without review | Parallel workers can create conflicting diffs; master must require review/merge readiness. |
| Desktop visual plugin on day one | No public Desktop/Web plugin surface exists. Use server tools and optional TUI plugin first. |
| Remote/cloud workers in v1 | Start with local git worktrees. Remote adapters can come later. |

## Architecture

`multitask` should be a dual-surface package:

| Surface | Required? | Role |
| --- | --- | --- |
| Server plugin | Yes | Core orchestration, tools, state, events, workspace/session API calls, safety checks. |
| TUI plugin | Optional | Command palette, dialogs, worker status panel, jump-to-worker actions. |
| Desktop/Web UI | No for v1 | Desktop users can still invoke server tools from chat; no custom visual surface. |
| External CLI | Optional later | Batch launch, CI integration, recovery/cleanup scripts. |

Package shape should follow OpenCode plugin constraints: split server and TUI entrypoints; do not put both `server` and `tui` in one default plugin object.

## Control Plane

### Master Session

The master session is the only place where the user gives the global goal and approves fan-out. It owns:

| Responsibility | Detail |
| --- | --- |
| Decomposition | Turn user goal into worker tasks with file scopes and dependencies. |
| Admission control | Enforce max workers, max cost/time, allowed paths, and risk classes. |
| Workspace/session creation | Create one workspace plus one child session per admitted worker. |
| Report aggregation | Collect reports and produce a single master status/merge plan. |
| Human escalation | Ask questions when workers conflict, block, or need permission. |

### Worker Session

Each worker session receives one bounded task and one worktree-backed workspace.

Worker prompt contract:

| Required Section | Purpose |
| --- | --- |
| Mission | One clear task, no side quests. |
| Scope | Allowed files/directories and explicitly forbidden files. |
| Dependencies | Upstream worker outputs that must be respected. |
| Constraints | No push, no merge to main, no modifying `.multitask/`; submit reports through the assigned worker report channel. |
| Verification | Commands/tests expected for this worker. |
| Report format | Structured final report required before stopping. |

Final report contract:

| Field | Required | Meaning |
| --- | --- | --- |
| `status` | Yes | `done`, `blocked`, `failed`, `needs-review`. |
| `summary` | Yes | Short human-readable result. |
| `workspaceID` | Yes | OpenCode workspace ID. |
| `sessionID` | Yes | Worker session ID. |
| `worktree` | Yes | Local worktree path. |
| `branch` | Yes | Worker branch if branch-backed. |
| `baseRevision` | Yes | Git revision used as the worker base. |
| `headRevision` | Yes | Worker branch HEAD if committed, or `uncommitted` if work remains only in the worktree. |
| `filesChanged` | Yes | Relative paths changed. |
| `diffRef` | Yes | Path or command for reviewing the worker diff/patch. |
| `testsRun` | Yes | Commands and pass/fail result. |
| `blockers` | Yes | Anything master/user must resolve. |
| `mergeNotes` | Yes | How to integrate safely. |

## Workspace Strategy

Prefer built-in OpenCode worktree adapter for v1:

| Step | API/Behavior |
| --- | --- |
| Verify support | `experimental.workspace.adapter.list` should include `worktree`. |
| Create workspace | `experimental.workspace.create` with type `worktree`, branch `multitask/<run>/<worker>` or detached fallback. |
| Wait for readiness | For local `worktree`, do not treat `workspace.status=connected` as sufficient. Use it as a connection signal, then wait for `worktree.ready` or verified worktree bootstrap completion before dispatching prompts. |
| Create child session | `session.create` with body `workspaceID`, `parentID`, title, metadata, and worker permission rules; route the request with workspace/directory context when using raw HTTP/SDK parameters. |
| Send task | `session.promptAsync` with `sessionID`, workspace/directory context, and worker prompt parts. |
| Keep visible | `experimental.workspace.syncList` after creation and after major lifecycle events. |

Hard requirement: launch OpenCode with `OPENCODE_EXPERIMENTAL_WORKSPACES=true` or `OPENCODE_EXPERIMENTAL=true` for workspace switching/sync. Without it, the plugin should fail clearly or degrade to plain worktrees only if explicitly configured.

### Branch Naming

Use deterministic branch/worktree names:

| Entity | Pattern |
| --- | --- |
| Run ID | `mt-YYYYMMDD-HHMMSS-shortgoal` |
| Worker ID | `w01`, `w02`, `w03` |
| Branch | `multitask/<run-id>/<worker-id>` |
| Session title | `multitask <run-id> <worker-id>: <short task>` |

Do not rely on model-generated names for file paths or branch names. Slug and length-limit every value.

## Plugin Tools

The model-facing API should be small. Avoid exposing low-level primitives like raw workspace creation to arbitrary agents.

| Tool | Who Calls It | Purpose |
| --- | --- | --- |
| `multitask_start` | Master session only | Validate plan, create run, create workspaces/sessions, dispatch prompts. |
| `multitask_status` | Master or user | Show run and worker states. |
| `multitask_collect` | Master session | Aggregate final reports and diffs. |
| `multitask_cancel` | User/master | Stop selected workers and mark run cancelled. |
| `multitask_cleanup` | User/master | Remove completed workspaces/worktrees after merge or discard. |
| `multitask_merge_plan` | Master session | Produce safe integration order and conflict list; does not merge automatically in v1. |

Tool permission policy:

| Rule | Reason |
| --- | --- |
| Only master session can call lifecycle tools by default. | Prevent recursive swarms. |
| Worker sessions cannot call `multitask_start`. | Avoid runaway nested workspace creation. |
| Worker sessions cannot remove workspaces. | Cleanup is coordinator/user-owned. |
| Worker sessions get scoped edit permissions. | Reduces parallel conflict blast radius. |
| Dangerous shell commands stay denied or ask. | Worktree isolation is not a security boundary. |

Enforcement requirements:

| Check | Where To Enforce |
| --- | --- |
| Caller role | `multitask_*` tool handlers must verify `ctx.sessionID` or input `sessionID` against durable run state before doing lifecycle work. Do not rely on prompt instructions. |
| Worker identity | Store `sessionID`, `workspaceID`, `parentID`, worker ID, and role in run metadata; reject calls when they do not match the current run. |
| Scoped writes | Enforce with permission config and/or `tool.execute.before` path checks, not only worker prompt wording. |
| Cleanup/merge authority | Require master/user-owned tool calls and completed/blocked run state before removing workspaces or producing integration instructions. |

## State Model

State must survive OpenCode restart and worker session compaction.

Preferred storage:

| Store | Location | Notes |
| --- | --- | --- |
| Runtime DB | OpenCode/plugin data dir | Authoritative durable run/worker/session/workspace state; not intended for git. |
| Project audit export | `.multitask/` in the master workspace | Optional human-readable export for reports and merge plans; auto-add to `.git/info/exclude` or ask before committing. |
| Worker reports | Runtime DB first; optional `.multitask/runs/<run-id>/reports/<worker-id>.md` export | Avoid treating each worker worktree as an independent source of truth. Master collection owns the export. |
| Event log | Runtime DB first; optional `.multitask/runs/<run-id>/events.jsonl` export | Append-only, useful for recovery/debug, but plugin state remains authoritative. |

Minimum run state:

| Field | Meaning |
| --- | --- |
| `runID` | Stable run identifier. |
| `masterSessionID` | Coordinator session. |
| `goal` | Original user goal. |
| `workers[]` | Worker IDs with task, scope, workspaceID, sessionID, branch, base/head revisions, diffRef, status. |
| `createdAt/updatedAt` | Recovery and cleanup. |
| `maxParallel` | Admission limit used. |
| `status` | `planning`, `running`, `collecting`, `done`, `cancelled`, `failed`. |

## Scheduling And Conflict Policy

V1 should be conservative.

| Rule | Default |
| --- | --- |
| Max workers | 2-4 unless user explicitly raises it. |
| Require file scopes | Yes. No scope means serial or ask user. |
| Global files | Serialize tasks touching package managers, lockfiles, root config, auth/security. |
| Same file overlap | Do not run in parallel. |
| Parent/child directory overlap | Do not run in parallel unless read-only. |
| Dependencies | Use DAG waves; only dispatch next wave after dependencies report done. |
| Dirty main worktree | Ask before starting, or require clean base. |

Borrow concepts from `opencode-swarm` Lean Turbo lane planning, but implement only what is needed for workspace/session fan-out.

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
| “Run this with multitask” | Master proposes worker plan and asks for approval. |
| “Status” | Master calls `multitask_status` and summarizes workers. |
| “Collect results” | Master calls `multitask_collect`, reads reports/diffs, produces merge plan. |
| “Cancel worker 2” | Master calls `multitask_cancel` for that worker. |

### TUI Plugin UX

Optional but valuable:

| TUI Feature | Purpose |
| --- | --- |
| Command palette `Multitask: Start` | Opens a prompt/dialog for goal and max workers. |
| Sidebar/status slot | Shows worker count, running/done/blocked. |
| Worker list dialog | Jump to worker session/workspace. |
| Toasts | Workspace creation, worker done/blocked/failure. |

Use current TUI keymap API (`api.keymap.registerLayer`). TUI code may invoke server-side `multitask_*` tools/actions or read status, but it should not duplicate workspace/session creation logic or bypass server-side state and safety checks. Do not use deprecated `api.command.*`.

### Desktop/Web UX

No custom Desktop/Web visual plugin for v1. Desktop sessions can still use server tools and worker sessions because Desktop talks to the same OpenCode server/sidecar. If Desktop-specific UI is required later, it needs a core `packages/app` / `packages/desktop` change or future Desktop extension point.

## Failure Modes

| Failure | Handling |
| --- | --- |
| Experimental workspace flag missing | Fail clearly before creating anything, unless user enables degraded mode. |
| Adapter unavailable | Tell user `worktree` adapter not listed; do not manually shell out unless configured. |
| Workspace create fails | Mark worker failed, keep run active if other workers can proceed. |
| Workspace/worktree readiness timeout | Retry status/readiness polling, then mark blocked; do not send prompt blindly. |
| Child session create fails | Remove workspace if empty and mark worker failed. |
| Prompt dispatch fails | Keep workspace/session for inspection; mark failed. |
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
| State writes | Workers cannot modify `.multitask/`; master/plugin collection owns audit exports. |
| Recursive spawning | Workers cannot call `multitask_start`; master-only lifecycle tools. |
| External directories | Deny or ask; worktree path should be the trusted root. |
| Cost runaway | Max workers, max runtime, max tool calls, max retries. |

## Open Questions For Product Design

These are not blockers for the design doc, but should be decided before implementation:

1. Should `multitask` be a standalone plugin or an `opencode-forge` contribution/mode?
2. Should workers be branch-backed by default, or detached worktrees until merge time?
3. Should worker prompts be generated by the master model or deterministic from a user-approved plan file?
4. Should v1 support Desktop users only through chat tools, or also require TUI for the full workflow?
5. Should `.multitask/` audit exports be generated automatically, only on demand, or disabled by default?
6. Should `multitask` integrate with existing `opencode-swarm`/`swarm-tools` file reservation formats, or keep its own minimal scope model?

## MVP Plan

| Phase | Deliverable | Exit Criteria |
| --- | --- | --- |
| 0. Spike | Prove workspace/session fan-out with raw SDK calls. | One master action creates two worktree workspaces, two sessions, and sends async prompts. |
| 1. Server MVP | Server plugin tools and durable state. | `multitask_start/status/collect/cancel` work from chat. |
| 2. Safety MVP | Scope validation and permission rules. | Workers cannot edit outside assigned scope or start nested multitask runs. |
| 3. TUI MVP | Command palette/dialog/status/jump. | User can launch, monitor, and jump to worker sessions from TUI. |
| 4. Recovery | Restart/reconcile. | Plugin can recover active runs after OpenCode restart. |
| 5. Merge Assist | Aggregated diff and merge plan. | Master produces deterministic integration order and conflict report. |

## Verification Checklist

Use a test repo with a clean git status.

1. Start OpenCode with `OPENCODE_EXPERIMENTAL_WORKSPACES=true`.
2. Confirm `experimental.workspace.adapter.list` includes `worktree`.
3. Start a run with two independent tasks and non-overlapping scopes.
4. Confirm two workspaces are created, reach `connected`, and for worktrees complete `worktree.ready` or equivalent bootstrap verification.
5. Confirm two sessions are created with `workspaceID`, `parentID`, and correct workspace/directory routing.
6. Confirm each worker receives its prompt through `promptAsync` and begins running.
7. Confirm `multitask_status` reports running/done/blocked accurately.
8. Force one worker failure and confirm the run remains inspectable.
9. Force one permission/question blocker and confirm it surfaces to master/user.
10. Confirm `multitask_collect` reads reports and produces a merge plan.
11. Confirm `multitask_cleanup` removes only selected completed workspaces.
12. Restart OpenCode mid-run and confirm recovery/reconciliation.

Relevant upstream tests to study or mirror in the future project:

| Test Area | Upstream Test Names From Source Search |
| --- | --- |
| Workspace adapter | `packages/opencode/test/plugin/workspace-adapter.test.ts` |
| HTTP workspace API | `packages/opencode/test/server/httpapi-workspace.test.ts` |
| Workspace routing | `packages/opencode/test/server/workspace-routing.test.ts` |
| TUI workspace create | `packages/opencode/test/cli/cmd/tui/dialog-workspace-create.test.ts` |

## Decision

Build `multitask` only if the exact fan-out/fan-in workflow is required and cannot be contributed to `opencode-forge`. The project should be intentionally thin: use OpenCode's workspace/session APIs, reuse proven patterns from Forge and Swarm, and focus on master-session orchestration plus report aggregation.
