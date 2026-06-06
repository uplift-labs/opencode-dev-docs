# Plugin Runtime

Справочник по тому, как OpenCode находит, устанавливает и исполняет plugins.

Проверено по `packages/plugin/src/index.ts`, `packages/plugin/src/tui.ts`, `packages/opencode/src/plugin/index.ts`, `loader.ts`, `shared.ts`, `config/plugin.ts`.

## Где Лежат Plugins

Официально поддерживаются:

| Scope | Путь |
| --- | --- |
| Project | `.opencode/plugins/` |
| Global | `~/.config/opencode/plugins/` |
| Config | `opencode.json` field `plugin` |

По исходникам есть дополнительная совместимость:

| Возможность | Деталь |
| --- | --- |
| Singular dirs | Сканируются и `plugin/`, и `plugins/`. |
| Extensions | Local plugin scan ищет `*.ts` и `*.js`. |
| Path specs | `plugin` config может указывать `./plugin.ts`, `./plugin`, absolute path или `file://`. |
| Options | Config item может быть строкой или tuple `[specifier, options]`. |

Пример с options:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": [
    "opencode-wakatime",
    ["./.opencode/plugins/guard.ts", { "strict": true }]
  ]
}
```

В plugin:

```ts
export default {
  id: "uplift.guard",
  server: async (ctx, options) => {
    const strict = options?.strict === true
    return {}
  },
}
```

## Порядок Загрузки

Официальный порядок hook execution:

| Order | Source |
| --- | --- |
| 1 | Global config `~/.config/opencode/opencode.json` |
| 2 | Project config `opencode.json` |
| 3 | Global plugin directory `~/.config/opencode/plugins/` |
| 4 | Project plugin directory `.opencode/plugins/` |

Config files merged, not replaced. Local path specs are resolved relative to the config file that declared them, not relative to process cwd.

NPM dedupe works by package name. Local file plugins dedupe by exact file URL. If the same package appears in multiple configs, the later config wins by OpenCode merge/dedupe logic.

## Module Shapes

### Recommended Server Plugin

```ts
import type { Plugin } from "@opencode-ai/plugin"

export default {
  id: "uplift.server-plugin",
  server: async (ctx, options) => {
    return {
      event: async ({ event }) => {},
    }
  },
} satisfies { id: string; server: Plugin }
```

### Recommended TUI Plugin

```ts
import type { TuiPlugin } from "@opencode-ai/plugin/tui"

export default {
  id: "uplift.tui-plugin",
  tui: async (api, options, meta) => {
    api.ui.toast({ message: `Loaded ${meta.id}`, variant: "success" })
  },
} satisfies { id: string; tui: TuiPlugin }
```

### Legacy Server Plugin

Server loader still accepts legacy named exports where every export is a plugin function or object with `server`:

```ts
export const MyPlugin = async (ctx) => ({})
```

Prefer default object for new work. Local path plugins in the new object format must export a non-empty `id`.

## Server И TUI В Одном NPM Package

Default plugin object cannot contain both `server` and `tui`; loader rejects objects with both fields. For an NPM package, ship separate entrypoints:

```json
{
  "name": "opencode-uplift-plugin",
  "type": "module",
  "exports": {
    "./server": "./dist/server.js",
    "./tui": "./dist/tui.js"
  },
  "engines": {
    "opencode": ">=1.0.0 <2"
  }
}
```

Entrypoint resolution from source:

| Kind | NPM resolution | File/directory resolution |
| --- | --- | --- |
| Server | `exports["./server"]`, else `main` | exact file, directory `index.ts`, `index.tsx`, `index.js`, `index.mjs`, `index.cjs` when package/directory rules allow it |
| TUI | `exports["./tui"]` | exact file or directory index for file plugins |

NPM plugins can declare `engines.opencode`. If current OpenCode version does not satisfy the range, OpenCode skips the plugin.

## Plugin Context

Server plugin receives:

| Field | Meaning | Notes |
| --- | --- | --- |
| `client` | Type-safe OpenCode SDK client | Talks to the in-process server with auth headers. Good for logs, sessions, TUI control. |
| `project` | Current project info | From server project state. |
| `directory` | Current session/project directory | Prefer this over `process.cwd()`. |
| `worktree` | Git worktree root | Useful for stable relative paths. |
| `serverUrl` | Current server URL | Getter; can be random TUI port or default. |
| `$` | Bun shell API | Available when running under Bun runtime. |
| `experimental_workspace.register` | Register workspace adapter | Hidden/experimental, see `05-hidden-capabilities.md`. |

TUI plugin receives:

| Field | Meaning |
| --- | --- |
| `api.keymap` | Current TUI command/keybinding API: register layers, dispatch commands, add bindings. |
| `api.command` | Deprecated v1 compatibility shim for register/trigger/show TUI commands. |
| `api.route` | Register and navigate routes. |
| `api.ui` | Dialogs, prompt component, toast, slots. |
| `api.keybind` | Match/print/create keybinds. |
| `api.tuiConfig` | Frozen view of TUI config. |
| `api.kv` | Plugin-visible persistent-ish key/value state. |
| `api.state` | Read-only live TUI state: sessions, parts, LSP, MCP, path, provider. |
| `api.theme` | Theme inspection/set/install. |
| `api.client` | V2 SDK client. |
| `api.event` | Subscribe to typed events. |
| `api.renderer` | OpenTUI renderer. |
| `api.slots` | Register Solid slot plugins. |
| `api.plugins` | List/activate/deactivate/add/install plugins. |
| `api.lifecycle` | AbortSignal and dispose hooks. |

## Hook Execution Semantics

| Hook type | Execution | Error behavior |
| --- | --- | --- |
| Trigger hooks like `tool.execute.before` | Sequential, in plugin load order | Error propagates to caller and can block/abort the action. |
| `config` hook | Called once after plugin initialization | Errors are logged and ignored per hook. |
| `event` hook | Called for every bus event | Fire-and-forget; OpenCode does not await it. |
| TUI plugin cleanup | Reverse order, 5s total timeout | Errors/timeouts are logged. |

Implication: use `event` for telemetry and async side effects, but do not rely on it to block behavior. Use trigger hooks for guardrails.

## Dependencies

Local plugins and tools can use dependencies from the config directory package:

```json
{
  "dependencies": {
    "shescape": "^2.1.0"
  }
}
```

OpenCode waits for dependency installation before loading local plugins/tools that need it. NPM plugins are installed into OpenCode cache via Bun/NPM helper.

For a full local project layout with `opencode.json`, `.opencode/package.json`, plugin file, custom tool and verification loop, see [`00-quickstart.md`](00-quickstart.md).

## Special Flags And Modes

| Flag/env | Effect |
| --- | --- |
| `OPENCODE_PURE` | Skips external plugins. Internal auth plugins still load. |
| `OPENCODE_CONFIG_DIR` | Adds a custom config directory with agents/commands/plugins/tools/etc. |
| `OPENCODE_CONFIG` | Loads a custom config file. |
| `OPENCODE_CONFIG_CONTENT` | Highest standard inline config override. |
| `OPENCODE_SERVER_PASSWORD` | Enables basic auth for `opencode serve`/`opencode web`. |

## Logging From Plugins

Prefer structured logs through SDK:

```ts
await client.app.log({
  body: {
    service: "uplift.guard",
    level: "info",
    message: "blocked tool call",
    extra: { tool: input.tool },
  },
})
```

`console.log` works, but `client.app.log()` integrates with OpenCode logging and keeps plugin output cleaner.
