# Desktop Development

Практическая карта для изменений в OpenCode Desktop и Web UI.

Короткий ответ: UI/UX Desktop менять принципиально можно, потому что OpenCode open source и Desktop app живет в monorepo. Но это не тот же extension surface, что TUI plugins. TUI slots вроде `home_prompt_right` и `session_prompt_right` работают в terminal UI, а не в Electron/Web renderer. Для Desktop UI нужен change в core app code или отдельный публичный Desktop/Web extension point, если он появится позже.

## Architecture

| Package | Role | When To Touch |
| --- | --- | --- |
| `packages/app` | Shared SolidJS web UI used by browser/web and Desktop renderer. | Most chat, prompt, session, settings, status, model picker and layout UI changes. |
| `packages/desktop` | Electron shell around `packages/app`: main/preload/renderer bootstrap, native menus, sidecar, updater, Windows/WSL integration. | Native window behavior, IPC, menus, notifications, sidecar startup, packaging. |
| `packages/ui` | Shared design system components. | Reusable primitives, icons, popovers, buttons, theme tokens. |
| `packages/opencode` | Core server, session loop, tools, auth, providers, HTTP API. | New backend data or behavior needed by UI. |
| `packages/sdk` | Generated clients/types for the HTTP API. | Regenerate after changing API routes or schemas. |
| `packages/plugin` | Server/TUI plugin types. | User-land plugin APIs; TUI plugin types do not automatically affect Desktop. |

Desktop renderer imports `@opencode-ai/app` and provides a Desktop `PlatformProvider` implementation through `packages/desktop/src/renderer/index.tsx`. Renderer code should call only `window.api` from preload. Native operations should be exposed through preload and implemented in main-process IPC.

## Source Of Truth

Official docs currently cover Web, config, server, SDK, permissions, custom tools, commands, keybinds and TUI config, but `https://opencode.ai/docs/desktop` returns 404. Treat Desktop behavior as source-backed, not prose-backed.

| Claim | Verify In |
| --- | --- |
| Desktop renderer is the shared app UI. | `packages/desktop/src/renderer/index.tsx`, `packages/app/src/*`. |
| Desktop starts and talks to a local sidecar server. | `packages/desktop/src/main/server.ts`, `packages/desktop/src/main/sidecar.ts`, `packages/desktop/src/renderer/index.tsx`. |
| Native Desktop operations go through preload IPC. | `packages/desktop/src/preload/types.ts`, `packages/desktop/src/preload/index.ts`, `packages/desktop/src/main/ipc.ts`. |
| Server plugins, custom tools, permissions and hooks are available to Desktop sessions. | `packages/opencode/src/plugin/*`, `packages/opencode/src/tool/registry.ts`, official `plugins`, `custom-tools`, `permissions` docs. |
| TUI plugins are terminal UI extensions, not Desktop/Web renderer extensions. | `packages/plugin/src/tui.ts`, `packages/opencode/src/cli/cmd/tui/plugin/runtime.ts`, `packages/app/src/*`. |
| `question` tool is enabled for Desktop client flows. | `packages/opencode/src/tool/registry.ts` checks `flags.client` for `"app"`, `"cli"`, `"desktop"`. |
| `/tui/*` routes publish or consume TUI events. | `packages/opencode/src/server/routes/instance/httpapi/groups/tui.ts`, `packages/opencode/src/cli/cmd/tui/event.ts`, `packages/opencode/src/cli/cmd/tui/app.tsx`. |

## Extension Surface Decision

| Goal | Best Path | Notes |
| --- | --- | --- |
| Add a compact status near the terminal prompt in TUI | TUI plugin slot such as `home_prompt_right` or `session_prompt_right`. | Source-only/internal surface; pin and smoke test versions. |
| Add a compact status in Desktop/Web composer | Core UI change in `packages/app`, usually near `components/prompt-input.tsx` or existing status components. | Local TUI plugins will not render there. |
| Add richer Desktop/Web status details | Prefer existing popover/dialog patterns in `packages/app`, for example `StatusPopover` style. | Keep the always-visible surface short. |
| Add provider/model/session data | Prefer existing SDK queries/context in `packages/app`; add a server route only if data is not already exposed. | After API changes, run SDK generation. |
| Add OS-native behavior | Add IPC in `packages/desktop/src/main/ipc.ts`, expose it in `src/preload`, consume through `window.api`. | Do not access Electron APIs directly from app renderer components. |
| Add policy, tools, permissions, auth or provider behavior | Server plugin, custom tool, permission config, or core server change. | These work for Desktop because Desktop talks to an OpenCode server/sidecar. |
| Trigger TUI behavior from another process | HTTP/SDK `/tui/*` endpoints. | Controls a running terminal TUI only; not a Desktop plugin API. |

## Desktop Vs TUI Transfer Matrix

| Console/TUI Improvement | Transfers To Desktop? | Practical Desktop Path |
| --- | --- | --- |
| Server plugin hooks such as `tool.execute.before`, `chat.params`, `chat.headers`, `tool.definition`, `auth`, `provider` | Yes | Same server plugin runs behind Desktop sidecar. Validate with Desktop client because UX differs. |
| Local custom tools in `.opencode/tools/*.ts` | Yes | Model can call them from Desktop sessions through the same tool registry. |
| Permission rules and custom tool `ctx.ask(...)` | Yes | Desktop shows permission/question UX through app session state. Keep prompts concise. |
| Built-in `question` tool | Yes | Enabled for `desktop` client; use for agent-run choices instead of custom Desktop UI when possible. |
| MCP/config/provider/auth integration | Yes | Configure server side; Desktop consumes resulting sessions/providers/tools. |
| TUI commands, routes, slots, dialogs, themes, keymap layers | No | Rebuild as `packages/app`/`packages/desktop` UI or propose a Desktop/Web extension point. |
| TUI `/tui/append-prompt`, `/tui/execute-command`, `/tui/publish`, `/tui/show-toast` | No for Desktop UI | Useful for automating terminal TUI. `/tui/execute-command` maps legacy aliases; `/tui/publish` can send `tui.command.execute` for current command names. |
| TUI `tui.json` keybind/theme behavior | Mostly no | Desktop/Web has separate app UI and platform settings; verify each setting before documenting transfer. |

## Desktop-Friendly UX Without Core UI PRs

Prefer these when the goal is better Desktop behavior but not new Desktop chrome:

| Need | Use |
| --- | --- |
| Guard risky operations | `tool.execute.before`, permission config, or custom tool `ctx.ask(...)`. |
| Give model a safe capability | Custom tool in `.opencode/tools/*.ts` or plugin `tool`. |
| Ask the user during an agent run | Built-in `question` tool. |
| Adjust model/provider behavior | `chat.params`, `chat.headers`, `auth`, `provider.models`, config. |
| Observe and report state externally | Server plugin `event`, SDK `client.event.subscribe()`, HTTP `/event`. |
| Open external dashboards or local helpers | Custom tool or hook that returns links/status; avoid blocking prompt submission. |

If the improvement requires persistent visual placement, composer widgets, status chips, command palette entries, settings pages, or native menus, treat it as a core UI change.

## Local Development

From the OpenCode monorepo root:

```sh
bun install
```

Run the full Desktop app:

```sh
bun run --cwd packages/desktop dev
```

For fast shared UI iteration, run backend and app separately:

```sh
bun run --cwd packages/opencode --conditions=browser ./src/index.ts serve --port 4096
bun run --cwd packages/app dev -- --port 4444
```

Then open `http://localhost:4444`. The app dev server targets the backend on `localhost:4096` by default. Do not rely on `opencode dev web` for local UI/CSS changes, because it can proxy the hosted app instead of your local `packages/app` build.

Build/package Desktop:

```sh
bun run --cwd packages/desktop build
bun run --cwd packages/desktop package
```

Useful checks:

```sh
bun run --cwd packages/app typecheck
bun run --cwd packages/app test:unit
bun run --cwd packages/desktop typecheck
```

If you change server API or SDK-generated types, run:

```sh
./script/generate.ts
```

## Example: Usage Limits In The Composer

For a feature like showing weekly and 5-hour usage limits in the lower-right area of the Desktop composer:

| Concern | Recommendation |
| --- | --- |
| UI placement | Start in `packages/app/src/components/prompt-input.tsx`, near the composer toolbar/right controls. If details need expansion, use a tooltip or popover instead of long inline text. |
| Existing patterns | Reuse `StatusPopover`, `SessionContextUsage`, `ModelSelectorPopover`, `Tooltip`, `ProgressCircle`, and app design tokens before adding new primitives. |
| Data fetching | Fetch in a hook/context or TanStack Query layer, not inside JSX render. Cache briefly, dedupe in-flight requests, and keep the last known value on failure. |
| Backend source | Prefer existing SDK/client data. If no endpoint exists, add a server route in `packages/opencode`, update schemas/types, regenerate SDK, then consume from `packages/app`. |
| Desktop specifics | If the data requires native credentials, filesystem access, or sidecar state, expose a narrow `window.api` method via preload/main IPC. Otherwise keep it in shared app code so Web and Desktop stay consistent. |
| Failure behavior | This is decorative UI: fail open, hide the indicator or show stale value, and never block prompt submission. |
| Responsive behavior | On narrow widths, collapse to an icon/progress chip with a tooltip. Do not push model/agent/worktree controls off-screen. |

Do not implement this as a TUI plugin if the target is the Desktop screenshot. A TUI plugin can solve the terminal prompt-adjacent version, but Desktop needs `packages/app` renderer code or a future Desktop/Web plugin API.

## Desktop/Web Extension Backlog

There is no public Desktop/Web plugin surface today. If designing one upstream, keep it narrower than the TUI API and source it from existing app seams.

| Candidate Extension Point | Why It Helps | Source Area To Prototype |
| --- | --- | --- |
| Composer status contribution | Lets plugins expose short non-blocking state without replacing prompt UI. | `packages/app/src/components/prompt-input.tsx` and existing status/popover components. |
| Command palette action registry | Brings Desktop parity for local actions without TUI keymap leakage. | App command/menu components plus server-side plugin metadata route if user-land plugins contribute actions. |
| Notification/toast bridge | Gives server hooks safe user-visible feedback in Desktop/Web. | Existing app notification patterns, server event stream, Desktop native notification IPC only when needed. |
| Settings contribution | Lets plugins document toggles/options without editing config by hand. | App settings routes/components and config schema. |
| Read-only session side panel slot | Supports dashboards/timelines without changing assistant messages. | Session layout components and event/SSE data. |

RFC guardrails:

| Guardrail | Reason |
| --- | --- |
| Server hooks remain authoritative for security. | UI extension points can disappear or fail; guards must still run server-side. |
| Renderer extensions must be sandboxed and async. | Desktop renderer cannot receive arbitrary Node/Electron access. |
| Contributions must degrade gracefully in Web. | `packages/app` is shared by Web and Desktop. |
| TUI API names should not be reused unless behavior matches. | Avoid implying that TUI slots/dialogs/keymaps render in Desktop. |

## Contribution Expectations

OpenCode upstream accepts bug fixes, provider/LSP/formatter work and docs readily, but UI or core product features need design review with the core team before implementation. For net-new Desktop UI:

| Step | Standard |
| --- | --- |
| Issue first | Open a focused issue describing the problem, target users, proposed placement and screenshots/mockups. |
| Design review | Wait for maintainer direction before investing in a feature PR. |
| Small PR | Keep the change narrow: data source, one compact component, one placement. |
| Screenshots/video | Include before/after evidence for Desktop and, if shared app UI changes, Web. |
| Verification | Document exact commands run and manually verify Windows behavior when the change touches Desktop-specific code. |

## Desktop UI Checklist

| Check | Standard |
| --- | --- |
| Shared-first | Put reusable UI in `packages/app`; use `packages/desktop` only for native wrapper concerns. |
| No renderer privilege leaks | Renderer components call platform abstractions or `window.api`; no direct Electron or Node APIs in shared UI. |
| No hot-path blocking | No sync filesystem, subprocess, or network calls in render/composer paths. |
| Query hygiene | Timeout external calls, dedupe requests, cache briefly, and cancel/ignore stale updates after unmount. |
| Theme/i18n | Use design tokens and translation keys; avoid hardcoded colors/text in reusable UI. |
| Multi-window | Shared caches and refreshes should not stampede across multiple Desktop windows. |
| Windows/WSL | Verify path handling and native dialogs on Windows, especially when WSL integration is enabled. |
| Upgrade drift | Recheck package structure, preload API and SDK route names before updating support notes. |
