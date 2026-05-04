# Plugin Best Practices

Практический стандарт для production-grade OpenCode plugins, TUI extensions, custom tools и guardrails.

Главная цель: максимум удобства, скорости, безопасности и надежности без ощущения, что plugin живет отдельно от OpenCode.

## Core Principles

| Principle | Meaning |
| --- | --- |
| Native first | Используйте существующие OpenCode primitives до собственных обходных путей: permissions, questions, events, SDK, TUI slots, renderer, `api.kv`, `api.state`. |
| Never block the first prompt | Первый user prompt должен уходить в модель сразу. Bootstrap, discovery, git/network reads и cleanup должны быть async/background, кроме операций, которые прямо защищают опасный tool call. |
| Fast by default | UI render path не должен запускать sync filesystem, sync child processes, long network calls или expensive git commands. |
| Fail closed for safety | Если plugin защищает writes/shell/secrets/sandbox boundaries и не может проверить безопасность, risky operation должна быть blocked/asked. |
| Fail open for decoration | Если plugin рисует status, badge, usage, toast или terminal title, ошибка должна убрать/сохранить последний UI, но не ломать prompt/TUI. |
| Respect user flow | Не открывайте модалки и permission prompts без причины. Используйте non-blocking status/toast для информации, permission/question только когда нужно решение пользователя. |
| Make drift visible | Hidden/internal APIs требуют feature flags, documented compatibility notes и smoke tests на upgrade. |

## Transferable Uplift Patterns

These patterns showed up repeatedly across `uplift-labs` tooling. Borrow them only when the OpenCode plugin has the same pressure; do not cargo-cult shell/tooling decisions into places where OpenCode already provides a better native surface.

| Pattern | Good To Borrow | OpenCode Application | Do Not Overcopy |
| --- | --- | --- | --- |
| Tool-agnostic core + thin adapter | Keep domain logic independent from one host's hook protocol. | Put policy/evaluation logic in plain TS modules; keep OpenCode plugin hooks as translators. | Do not add an abstraction layer for a one-off 30-line plugin. |
| Single stable public entry point | Make integration contracts obvious and testable. | Expose one plugin entrypoint and one documented custom tool/API surface; mark internals unstable. | Do not expose every helper as public API. |
| Hot-path router/multiplexer | Avoid starting many subprocesses or expensive checks for every command. | In one `tool.execute.before`, parse once, select relevant checks, short-circuit on block. | Not needed when checks are cheap in-process functions. |
| Early skip for irrelevant work | Safe commands should not pay for heavy guards. | Route `git status`/read-only flows away from LLM verification or filesystem scans. | Do not skip security checks based on weak heuristics when the action is actually risky. |
| Severity aggregation | Combine multiple checks predictably: `block` > `ask` > `warn` > pass. | If your plugin runs several guard checks, return the strongest result and keep warnings concise. | Do not bury a hard safety failure behind a nicer warning. |
| Idempotent install/config mutation | Re-running install/update must converge without duplicating hooks. | Stable plugin `id`, unique config markers, preserve unrelated `opencode.json` fields. | Avoid plugin code that rewrites user config on every startup. |
| Kill switches and per-feature toggles | Users need emergency disable and noise control. | `PLUGIN_DISABLED=1`, per-feature env flags, CI/test auto-disable when appropriate. | Do not hide safety-critical disablement; log or document what was disabled. |
| TP/TN fixture tests | Guard behavior needs regression tests for true positives and true negatives. | Fixture-test path rules, shell command guards, permission decisions and event parsers. | Do not rely only on happy-path snapshots. |
| Background learning loop | Capture session lessons while context is fresh, process later with review. | Use `event`/idle lifecycle to schedule non-blocking reflection/telemetry. | Never run long reflection/model calls inside prompt/tool hot path. |
| Public contract docs | State input, output, exit/failure semantics and stability promise. | For published plugins, document hook behavior, env vars, events consumed, effects produced. | Do not promise stability for hidden OpenCode internals you cannot control. |

## Hot Hook Aggregation

`singularity` optimized Codex Bash hooks by replacing several independent hook process launches with one router process. The router parses the payload once, selects only relevant checks, then runs matching upstream guards. This matters because hook startup cost is paid on every Bash command.

OpenCode implication:

| Situation | Recommendation |
| --- | --- |
| Your OpenCode plugin implements checks directly in JS/TS. | You already run inside one process. Use one `tool.execute.before` hook and dispatch to small functions. |
| Your plugin shells out to several existing guard CLIs. | Add a router/multiplexer. Parse args once, skip irrelevant checks, short-circuit on first block. |
| You have one rare expensive check. | Gate it by tool id, command shape and session state before starting it. |
| Checks are decorative or advisory. | Run in background or downgrade to warning; do not block the hot path. |
| Checks are the only safety boundary for writes/shell. | It is acceptable to wait, but every subprocess needs timeout and actionable failure. |

Minimal routing shape:

```ts
type Decision =
  | { severity: "pass" }
  | { severity: "warn"; message: string }
  | { severity: "ask"; message: string }
  | { severity: "block"; message: string }

const rank = { pass: 0, warn: 1, ask: 2, block: 3 } as const

async function runSelectedChecks(input: { tool: string }, args: any): Promise<Decision> {
  let best: Decision = { severity: "pass" }

  for (const check of selectChecks(input.tool, args)) {
    const next = await check(args)
    if (rank[next.severity] > rank[best.severity]) best = next
    if (next.severity === "block") return next
  }

  return best
}
```

Use cheap syntactic routing for performance. Do not use routing heuristics as the final safety proof. For example, it is fine to skip a commit-only verifier for `git status`; it is not fine to allow `bash` writes because a regex failed to notice redirection or a heredoc.

## Core/Adapter Split

Several uplift tools use a stable core protocol plus thin host adapters. For OpenCode plugins, the same architecture is useful when logic may be reused by a CLI hook, MCP server, SDK daemon, or TUI plugin.

| Layer | Responsibility |
| --- | --- |
| Core policy | Pure functions or small modules: classify command/path/event, compute decision, format user-facing reason. No OpenCode SDK calls. |
| OpenCode server adapter | Hook signatures, `ctx.ask`, `client.app.log`, `tool.execute.before/after`, config/options. |
| TUI adapter | Slots, dialogs, renderer, theme, `api.kv`, `api.state`, lifecycle cleanup. |
| External adapter | HTTP/SDK/SSE mapping, auth, long-running process lifecycle. |

Benefits:

| Benefit | Why It Matters |
| --- | --- |
| Testability | Core can be fixture-tested without launching OpenCode. |
| Portability | Same policy can power OpenCode, Codex, Claude Code, CI or pre-commit. |
| Smaller blast radius | OpenCode API drift stays in adapter layer. |
| Cleaner UX | Adapter can map the same decision to native OpenCode permission/question/dialog semantics. |

Skip this split for tiny plugins where abstraction would be heavier than the logic.

## Contract And Fixture Tests

For guard-like plugins, document and test the decision contract.

| Contract Item | Example |
| --- | --- |
| Inputs | Tool id, args shape, session id, cwd/worktree, relevant env/config. |
| Outputs | Throw/block, mutate args, ask permission, warning log, metadata patch. |
| Failure behavior | Fail closed for safety, fail open for decoration, timeout fallback. |
| Stability promise | Which ids/env vars/options are stable, which are internal. |
| Opt-outs | Global disable, per-feature disable, CI behavior. |

Fixture naming pattern:

| Prefix | Meaning |
| --- | --- |
| `tp-*` | True positive: plugin must trigger/block/warn. |
| `tn-*` | True negative: plugin must stay silent/allow. |

High-value fixture classes for OpenCode plugins:

| Area | Fixtures To Include |
| --- | --- |
| Shell guards | Safe read-only command, destructive command, redirection, heredoc, `bash -c`, Windows path. |
| Path guards | Relative path, absolute path, symlink/worktree path, external directory, `.env.example` exception. |
| Patch guards | Add/Update/Delete markers, path with spaces, multiple files, protected file. |
| Event parsers | Missing fields, old event shape, V2 event shape, unrelated session id. |
| TUI state | Missing renderer, disposed component, duplicate event burst, stale async result. |

## Reflection And Continuous Improvement

`reinforce` shows a useful pattern: capture lessons automatically, but process them later through a reviewed plan. This is valuable for plugin projects because UX and guard friction only become obvious after real use.

OpenCode-friendly version:

| Step | Implementation |
| --- | --- |
| Capture | On session idle/end, schedule a background summary of plugin errors, blocked actions, user rejects and latency spikes. |
| Accumulate | Store small structured records under plugin state/cache or a repo-local diagnostics directory. |
| Remind | When enough records accumulate, show a low-friction toast or command palette item. |
| Distill | Generate proposed improvements in a docs/issue file, not directly into runtime policy. |
| Review | User approves which changes become guard rules, UX tweaks or docs updates. |

Do not add self-improvement loops that silently change runtime safety policy. Reflection can propose; humans approve.

## Practices To Treat Carefully

Some good ideas in hook-heavy repos can become bad OpenCode plugin advice if applied blindly.

| Idea | Use Carefully Because | Good OpenCode Version |
| --- | --- | --- |
| Bash-only zero-dependency core | Great for portable shell hooks, but OpenCode plugins already run in JS/Bun and have SDK/TUI APIs. | Use TypeScript for plugins; shell out only for existing tools or platform-specific operations. |
| Fail-open guard multiplexer | Reasonable when guard infrastructure is optional and must never brick a CLI. | For the only safety boundary around writes/shell, fail closed with clear recovery. |
| Regex command classification | Fast for routing obvious cases. | Use for early skip only; enforce dangerous actions with stronger parsing, path normalization or permissions. |
| File manifest gates | Useful in large generated-tooling repos. | Use only when orphan files are a real maintenance problem; otherwise it adds friction. |
| Fresh LLM verification on commit/push | Strong for complex tasks, expensive for routine edits. | Recommend or run for multi-AC/high-risk work; avoid every small decorative plugin change. |
| Background reflection | Valuable over time. | Keep it non-blocking, bounded and opt-in/transparent. |

## UX And UI Fit

OpenCode plugin должен ощущаться как часть OpenCode.

| Need | Prefer | Avoid |
| --- | --- | --- |
| Show persistent compact status | TUI slots such as `home_prompt_right`, `session_prompt_right`, `sidebar_*`; `api.renderer` when available. | Transcript messages, repeated toasts, terminal escape sequences as first choice. |
| Ask user before a risky tool action | Permission config or custom tool `ctx.ask(...)`. | Custom dialog from server hook, blocking event hook, ad-hoc stdin prompt. |
| Ask agent-time requirement question | Built-in `question` tool. | Permission prompt for non-permission decisions. |
| Configure plugin settings | TUI `DialogSelect`/`DialogPrompt`, `api.kv`, config/env toggles. | Asking the model to remember plugin configuration in conversation. |
| Notify about completed/background work | `api.ui.toast(...)` or terminal title/bell with explicit opt-out. | Modal dialog for routine success. |
| External dashboard integration | SDK/HTTP endpoints and event stream. | Polling UI state from files when SDK events exist. |

UI quality checklist:

| Check | Standard |
| --- | --- |
| Theme integration | Use `api.theme.current` colors; do not hardcode colors except intentional terminal title/text fallbacks. |
| Density | Keep prompt/sidebar content short and stable. Long text belongs in dialogs/routes, not prompt-adjacent slots. |
| Motion/noise | Spinners and bells must be rate-limited and configurable. No animation faster than needed. |
| Opt-out | Any terminal side effect, sound, polling, or internal UI override needs an env/config opt-out. |
| Unsupported host | Auto-disable for `TERM=dumb`, missing renderer, non-TTY output, missing auth, or absent workspace data. |
| Native escape hatch | If OpenCode later adds a native surface, document how to migrate away from workaround UI. |

## Performance Rules

The slowest operation in a plugin becomes the user's perceived OpenCode speed.

| Rule | Why |
| --- | --- |
| No sync child processes in TUI/runtime paths | `execFileSync`, `spawnSync`, sync git commands and sync shell probes can freeze prompt rendering and first prompt handling. |
| No sync filesystem scans in render paths | TUI slots can re-render often. Resolve once async, cache briefly, then render from state. |
| Timeout every external operation | Git, network, auth reads, subprocesses and API calls need bounded time. Use conservative defaults and env overrides. |
| Deduplicate in-flight work | Multiple events often arrive together. Use single-flight promises for bootstrap, git branch reads, usage fetches and cache refreshes. |
| Debounce event bursts | `message.updated`, `session.status`, `file.watcher.updated`, `vcs.branch.updated` can be noisy. Batch refreshes. |
| Cache with TTL, not forever | Branch, auth, usage and expensive discovery can be cached briefly; clear cache on direct invalidation events. |
| Prefer watcher + polling fallback | Watchers are low-latency but unreliable across platforms. Polling catches missed events but must be tunable/disableable. |
| Cancel after dispose | Async refresh must check `disposed`/`stopped` before updating state or creating watchers. |
| Unref timers/watchers where possible | Plugin timers should not keep OpenCode alive after exit. |

Latency budget guidance:

| Path | Target |
| --- | --- |
| TUI slot render | Pure render from current state; no awaited work inside JSX render. |
| Event hook | Schedule work and return quickly unless the event itself is your only safe enforcement point. |
| `experimental.chat.system.transform` | Should not wait on long bootstrap; inject pending context if needed. |
| `tool.execute.before` | May wait when enforcing safety, but must have timeouts and clear error messages. |
| Decorative refresh | Best-effort; keep previous value on failure. |

## Safety And Reliability

Security-sensitive plugins have different failure semantics from UI plugins.

| Plugin Type | Failure Mode |
| --- | --- |
| Guard for writes/shell/secrets/sandbox | Fail closed: deny, ask, or throw with actionable message. |
| Prompt/tool description enhancer | Fail open: do not block if extra prompt context cannot be generated. |
| Status/telemetry/UI decoration | Fail open: keep last value or hide component. |
| Cleanup/lifecycle | Best-effort plus fallback: do not crash OpenCode exit; leave enough markers/logs for later cleanup. |

Guardrails:

| Rule | Detail |
| --- | --- |
| Enforce in `tool.execute.before` or permissions | `tool.definition` is advisory and can improve model behavior, but it is not enforcement. |
| Shape-check hook output | MCP/task/custom outputs can differ. Defensive code avoids breaking unrelated tools. |
| Block by semantics, not only strings | For paths, resolve absolute paths, normalize Windows paths and compare containment. For shell, prefer parsed/safe allowlists where possible. |
| Keep errors actionable | Say what was blocked, why, and what safer alternative to use. |
| Avoid global permanent approvals | `always` patterns should be human-readable and narrow. |
| Treat hidden APIs as volatile | Pin/test versions and document fallback behavior. |

## Bootstrap And Lifecycle

The hardest UX bugs happen before the user sees anything.

Good bootstrap pattern:

| Phase | Behavior |
| --- | --- |
| Plugin load | Register hooks/slots immediately. Do only cheap setup. |
| Session created | Start async bootstrap in background. Store single-flight promise per session. Return immediately. |
| First prompt/system transform | If bootstrap is not ready, inject short pending context instead of blocking. |
| First risky tool call | Await bootstrap if safety depends on it; if bootstrap fails, fail closed with clear error. |
| Session deleted/dispose | Mark pending bootstrap cancelled, run cleanup async/best-effort, ignore stale completion safely. |
| Process exit | Use detached/best-effort cleanup; do not rely only on synchronous exit work. |

Anti-patterns:

| Anti-pattern | Consequence |
| --- | --- |
| Creating sandbox/worktree synchronously during first `system.transform` | First prompt appears frozen or delayed. |
| Running lifecycle cleanup synchronously in `session.deleted` | UI stalls on delete/exit. |
| Multiple events starting identical bootstrap | Races, duplicate worktrees, unpredictable environment. |
| Async refresh updating after `onDispose` | Ghost UI updates, leaked watchers/timers. |

## Case Study: `worktree-sandbox`

Repository: https://github.com/uplift-labs/worktree-sandbox

Relevant fixes in history:

| Commit Theme | Lesson |
| --- | --- |
| `fix(opencode): async sandbox bootstrap` | Sandbox creation moved out of prompt-critical path. Session events start background bootstrap; tools that need sandbox safety wait on the same promise. |
| `fix(opencode): avoid UI blocking operations` | Sync git/filesystem/subprocess work was replaced with async operations, timeouts and best-effort cleanup. |
| `fix(opencode): reduce sandbox sidebar refresh load` | Sidebar refreshes were debounced and scoped to sandbox-relevant changes. |
| `fix(opencode): keep branch badge opt-in` | UI decoration that might add noise became opt-in instead of always-on. |

Useful pattern from that plugin:

| Problem | Production Pattern |
| --- | --- |
| First prompt needs sandbox instructions, but sandbox may not exist yet. | Add short pending system context: sandbox is preparing; use relative project paths. Do not wait in `system.transform`. |
| Write/shell tools must not touch main repo. | `tool.execute.before` awaits sandbox readiness and rewrites/guards paths. If sandbox init fails, write-capable tools fail closed. |
| TUI file/branch display needs git state. | Async git commands with timeout, `Promise.all` for independent diffs, debounce, watcher plus fallback polling. |
| Session deleted while bootstrap is still running. | Mark entry cancelled; late bootstrap completion cleans itself up and does not mutate active state. |
| Built-in OpenCode sidebar does not match sandbox semantics. | Do not mutate internal plugin state by default; custom sidebar behavior is opt-in/configurable. |

## Case Study: `ai-cli-status-bar`

Repository: https://github.com/uplift-labs/ai-cli-status-bar

OpenCode did not have a dedicated status line/footer surface for all desired signals, so the implementation used the closest native surfaces first.

| Need | Chosen Design | Lesson |
| --- | --- | --- |
| Prompt-adjacent status | TUI slots `home_prompt_right` and `session_prompt_right`. | Prefer native TUI slots before terminal hacks. |
| Terminal tab status | `api.renderer.setTerminalTitle(...)` when available; escape sequences only as fallback. | Use renderer/native API first, terminal side effects second. |
| Bells for waiting/done | Rate-limited BEL, configurable states and minimum interval. | Notification should help, not spam. |
| Usage data from network/API | Memory cache, shared filesystem cache, lock, failure backoff and request timeout. | Slow network must not stall UI or stampede across windows. |
| Branch status | Git cache, in-flight dedupe, watcher invalidation, polling fallback. | Event bursts should coalesce into one refresh. |
| Stuck working spinner | Stale timeout returns to non-working state when final idle event is missed. | UI needs recovery paths for missing events. |

Relevant fixes in history:

| Commit Theme | Lesson |
| --- | --- |
| `fix(opencode): reduce status plugin UI stalls` | Cache/dedupe branch and usage refreshes; add network timeout; avoid concurrent refresh storms. |
| `fix(opencode): avoid tab status UI stalls` | Prefer renderer title API, slow down spinner, add stale timeout, wrap terminal writes. |
| `fix(opencode): limit tab bells to waits` | Audible notifications should map to user-action states, not every internal completion event. |

## Review Checklist Before Shipping

Use this checklist for any OpenCode plugin that changes UI, tools, permissions or lifecycle.

| Area | Check |
| --- | --- |
| First prompt | Start OpenCode with cold caches and send a prompt immediately. It must not wait on plugin bootstrap unless the prompt triggers a risky tool. |
| TUI responsiveness | Interact with prompt, command palette, session switch and scrolling while plugin refreshes. No visible stalls. |
| Slow dependencies | Simulate slow git/network/filesystem. UI should keep last value or show pending state; safety hooks should timeout/fail closed. |
| Event bursts | Trigger rapid `message.updated`, `session.status`, file watcher and branch events. Refreshes should debounce/dedupe. |
| Dispose/reload | Disable/reload plugin during async refresh. No late state updates, leaked timers, duplicate slots or duplicate event handlers. |
| Multi-window | Run two OpenCode instances on same repo. Shared cache/locks should prevent stampedes and corrupt writes. |
| Cross-platform paths | Test Windows path casing/separators, symlinks/worktrees and external directories. |
| Terminal support | Check `TERM=dumb`, non-TTY, tmux/screen if using terminal title/bell. Feature should auto-disable or degrade. |
| Missing auth/config | Missing auth files, missing env vars and invalid config must not crash TUI. |
| Permission UX | `ask`, `always`, `reject` patterns are readable and narrow; reject feedback guides safer next action. |
| Error messages | Blocked safety operations explain exactly what to do next. Decorative failures stay silent or debug-only. |
| Upgrade drift | Recheck public types/source locations listed in [`05-hidden-capabilities.md`](05-hidden-capabilities.md) before bumping OpenCode support range. |

## Anti-Pattern Catalog

| Anti-pattern | Better |
| --- | --- |
| `execFileSync` or `fs.existsSync` in hot TUI path | Async call during setup/observer refresh, cached state in render. |
| Fetching network data on every render/event | TTL cache, single-flight request, failure backoff. |
| Every event immediately triggers expensive refresh | Debounce, queue one pending refresh, clear cache only on invalidation events. |
| Modal dialog for routine plugin state | Prompt-adjacent slot, toast, or persistent status. |
| Direct terminal escape sequences while renderer API exists | Use `api.renderer` first; escape fallback only with opt-out. |
| Hiding/deactivating internal OpenCode plugins by default | Add your own slot/route or make override opt-in. |
| Prompt-only safety policy | Pair `tool.definition` guidance with permission or `tool.execute.before`. |
| Decorative plugin throws from event handler | Catch and keep last state; debug log only when enabled. |
| Timer/watchers without cleanup | Store unsubscribers/timers/watchers and clean them in `api.lifecycle.onDispose` or Solid `onCleanup`. |
| Permanent global state without session key | Key state by session/worktree/project and clear on session deletion/dispose. |
