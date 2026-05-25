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

## Extension Surface Decision

| Goal | Best Path | Notes |
| --- | --- | --- |
| Add a compact status near the terminal prompt in TUI | TUI plugin slot such as `home_prompt_right` or `session_prompt_right`. | Source-only/internal surface; pin and smoke test versions. |
| Add a compact status in Desktop/Web composer | Core UI change in `packages/app`, usually near `components/prompt-input.tsx` or existing status components. | Local TUI plugins will not render there. |
| Add richer Desktop/Web status details | Prefer existing popover/dialog patterns in `packages/app`, for example `StatusPopover` style. | Keep the always-visible surface short. |
| Add provider/model/session data | Prefer existing SDK queries/context in `packages/app`; add a server route only if data is not already exposed. | After API changes, run SDK generation. |
| Add OS-native behavior | Add IPC in `packages/desktop/src/main/ipc.ts`, expose it in `src/preload`, consume through `window.api`. | Do not access Electron APIs directly from app renderer components. |
| Add policy, tools, permissions, auth or provider behavior | Server plugin, custom tool, permission config, or core server change. | These work for Desktop because Desktop talks to an OpenCode server/sidecar. |

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
