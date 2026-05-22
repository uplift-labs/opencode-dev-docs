# Hidden Capabilities

Недокументированные или слабо документированные возможности OpenCode, найденные в исходниках.

Проверено по `anomalyco/opencode@9f708e748af34cf63c0b1010c4a07ddab1b10ef6`.

## High-Leverage Summary

| Capability | Why It Matters | Source area |
| --- | --- | --- |
| TUI plugins | Можно расширять UI, routes, commands, slots, dialogs, themes. | `packages/plugin/src/tui.ts`, `cli/cmd/tui/plugin/runtime.ts` |
| Server `chat.*` hooks | Можно менять LLM params, headers, messages, system prompt. | `session/llm.ts`, `session/prompt.ts` |
| `tool.definition` | Можно менять tool descriptions/schema до отправки модели. | `tool/registry.ts` |
| `provider` hook | Можно динамически получать models для существующего provider. | `provider/provider.ts` |
| `auth` hook | Можно заменить/добавить auth methods для providers. | `provider/auth.ts`, internal auth plugins |
| `config` hook | Plugin может программно менять resolved config до provider init. | `plugin/index.ts` |
| Workspace adapters | Можно регистрировать custom workspace target/create/remove logic. | `experimental_workspace.register`, `control-plane/*` |
| V2 events `session.next.*` | Более детальная телеметрия agent loop/tool/text/reasoning/compaction. | `v2/session-event.ts`, SDK V2 types |
| TUI plugin manager API | Plugins могут устанавливать, активировать и деактивировать другие plugins. | `api.plugins.*` in TUI runtime |
| Package `oc-themes` | NPM TUI plugin может поставлять themes без TUI code. | `plugin/shared.ts`, TUI runtime |

## Stability Legend

| Label | Meaning | How To Use |
| --- | --- | --- |
| Stable official | Covered by official docs and public package types. | Safe default for plugins intended for broad distribution. |
| Typed but undocumented | Present in `@opencode-ai/plugin` or SDK types, but not fully described in official docs. | Useful for production if you pin/test OpenCode versions. |
| Source-only internal | Found in source/runtime behavior, not guaranteed by official docs or stable public docs. | Use behind feature flags or with a compatibility check. |
| Experimental flag | Requires `OPENCODE_EXPERIMENTAL*` or similar flag. | Treat as opt-in and expect drift. |
| Known landmine | Declared or visible, but behavior is surprising or incomplete. | Avoid as a primary integration point unless explicitly tested. |

## Capability Stability Matrix

| Capability | Stability | Recheck Before Upgrade |
| --- | --- | --- |
| Basic server plugins, `event`, `tool.execute.before/after`, `shell.env`, plugin `tool` | Stable official | `packages/plugin/src/index.ts`, official `plugins` docs. |
| Local custom tools in `.opencode/tools/*.ts` | Stable official | `packages/plugin/src/tool.ts`, `tool/registry.ts`, official `custom-tools` docs. |
| Permission config and `ctx.ask(...)` | Stable official | `permission/index.ts`, `tool/*`, official `permissions` docs. |
| `chat.message`, `chat.params`, `chat.headers`, experimental chat/system hooks | Typed but undocumented | `packages/plugin/src/index.ts`, `session/llm.ts`, `session/prompt.ts`, `session/processor.ts`. |
| `tool.definition` | Typed but undocumented | `tool/registry.ts`; verify output shape still has `description` and `parameters`. |
| `auth` and `provider` hooks | Typed but undocumented | `provider/auth.ts`, `provider/provider.ts`, internal auth plugins. |
| TUI plugins, routes, commands, slots, dialogs, themes | Source-only internal | `packages/plugin/src/tui.ts`, TUI plugin runtime, public TUI package types. |
| `oc-themes` package field | Source-only internal | `plugin/shared.ts`, TUI runtime theme installer. |
| Workspace adapters | Source-only internal | `experimental_workspace.register`, control-plane/workspace source. |
| `session.next.*` events and sync envelopes | Typed but undocumented | `v2/session-event.ts`, SDK V2 generated types, `/event` smoke test. |
| Experimental tool listing endpoints | Experimental flag/source endpoint | `GET /doc`, `GET /experimental/tool/ids`, `GET /experimental/tool?provider=<p>&model=<m>`. |
| `permission.ask` hook | Known landmine | Confirm `plugin.trigger("permission.ask", ...)` exists before using; otherwise use permission config/events. |

## Upgrade Recheck Checklist

Run this when moving to a new OpenCode version or when publishing a plugin for users on mixed versions:

| Check | Why |
| --- | --- |
| Compare `packages/plugin/src/index.ts` and `packages/plugin/src/tui.ts`. | Public hook/TUI signatures can move faster than docs. |
| Check `tool/registry.ts` and built-in `tool/*` files. | Tool ids, permissions and model-specific tool selection affect guards. |
| Check `permission/index.ts` and official `permissions` docs. | Permission keys and `ask`/`always` behavior are security-sensitive. |
| Check `server/routes/*` or `GET /doc`. | SDK method names can lag/rename while HTTP routes remain inspectable. |
| Smoke test `tool.execute.before`, one custom tool, one permission prompt and one question flow. | This catches loader, hook, tool schema and UX regressions quickly. |

## TUI Plugins

TUI plugin format:

```ts
import type { TuiPlugin } from "@opencode-ai/plugin/tui"

export default {
  id: "uplift.tui",
  tui: async (api, options, meta) => {
    api.command.register(() => [
      {
        title: "Uplift Status",
        value: "uplift.status",
        description: "Show Uplift guard status",
        category: "Uplift",
        onSelect: () => api.ui.toast({ message: "Uplift guards active", variant: "success" }),
      },
    ])
  },
} satisfies { id: string; tui: TuiPlugin }
```

Use TUI commands for local, immediate slash-style actions that must not call the
LLM. Server-side `command.execute.before` is part of the slash/custom command
prompt flow: it mutates the prompt that will be sent to the model. A TUI command
registered with `api.command.register` can run local code directly from the TUI,
show a toast/dialog, and return without creating an assistant turn.

TUI API surfaces:

| API | Use |
| --- | --- |
| `api.command.register(cb)` | Add command palette items and slash metadata. |
| `api.command.trigger(value)` | Trigger a command by value. |
| `api.command.show()` | Open command UI. |
| `api.route.register(routes)` | Add routes. |
| `api.route.navigate(name, params)` | Navigate to a route. |
| `api.ui.Dialog*` | Render alert/confirm/prompt/select dialogs. |
| `api.ui.Prompt` | Render OpenCode prompt component. |
| `api.ui.toast(input)` | Show toast. |
| `api.ui.Slot` | Render slot content. |
| `api.slots.register(plugin)` | Register OpenTUI Solid slot plugin. |
| `api.keybind.*` | Match/print/create keybind sets. |
| `api.kv.get/set` | Store plugin state. |
| `api.state.*` | Read sessions, parts, todo, permissions, questions, LSP, MCP, VCS. |
| `api.theme.install(jsonPath)` | Install a theme from plugin path. |
| `api.event.on(type, handler)` | Subscribe to typed events. |
| `api.plugins.*` | Manage plugins from plugin code. |
| `api.lifecycle.onDispose(fn)` | Cleanup on deactivate/reload. |

Host slots from `TuiHostSlotMap`:

| Slot | Props |
| --- | --- |
| `app` | `{}` |
| `home_logo` | `{}` |
| `home_prompt` | `{ workspace_id?, ref? }` |
| `home_prompt_right` | `{ workspace_id? }` |
| `session_prompt` | `{ session_id, visible?, disabled?, on_submit?, ref? }` |
| `session_prompt_right` | `{ session_id }` |
| `home_bottom` | `{}` |
| `home_footer` | `{}` |
| `sidebar_title` | `{ session_id, title, share_url? }` |
| `sidebar_content` | `{ session_id }` |
| `sidebar_footer` | `{ session_id }` |

TUI plugin lifecycle:

| Behavior | Detail |
| --- | --- |
| `id` required for file plugins | Local path TUI plugins without `id` fail. |
| Duplicate IDs fail | Runtime rejects duplicate TUI plugin id. |
| Enabled state persisted | Uses `plugin_enabled` in TUI KV/config overlay. |
| Cleanup timeout | Dispose handlers run in reverse order with 5s total timeout. |
| Internal TUI plugins | Home/sidebar/plugin-manager features are themselves TUI plugins. |
| Experimental V2 debug plugin | Enabled by `OPENCODE_EXPERIMENTAL_EVENT_SYSTEM`. |

## TUI Themes From Plugins

NPM plugin package can include `oc-themes`:

```json
{
  "name": "opencode-uplift-theme-pack",
  "oc-themes": ["./themes/uplift-dark.json", "./themes/uplift-light.json"]
}
```

Even if a package has no TUI entrypoint, runtime can still read `oc-themes` and install themes. Paths must be relative and stay inside plugin package directory.

## Provider Auth Override

Source tests confirm a user plugin can override built-in `github-copilot` auth by registering the same provider id.

```ts
export default {
  id: "uplift.custom-copilot-auth",
  server: async () => ({
    auth: {
      provider: "github-copilot",
      methods: [{ type: "api", label: "Uplift Copilot Token" }],
      loader: async () => ({ access: process.env.UPLIFT_COPILOT_TOKEN }),
    },
  }),
}
```

Useful for enterprise auth, token brokering, OAuth wrappers, and provider-specific credential normalization.

## Provider Dynamic Models

`provider.models` can replace/extend model list for an existing provider.

```ts
export default {
  id: "uplift.provider-models",
  server: async () => ({
    provider: {
      id: "openai",
      models: async (provider, ctx) => ({
        "uplift-router": {
          id: "uplift-router",
          name: "Uplift Router",
          providerID: "openai",
          api: provider.models["gpt-4.1"]?.api ?? provider.models[Object.keys(provider.models)[0]].api,
          capabilities: provider.models[Object.keys(provider.models)[0]].capabilities,
          cost: { input: 0, output: 0, cache: { read: 0, write: 0 } },
          limit: { context: 128000, output: 4096 },
          options: {},
        } as any,
      }),
    },
  }),
}
```

The hook only runs if provider exists in models.dev/OpenCode database and is not disabled.

## Config Mutation Hook

`config(cfg)` runs before providers are initialized. This is powerful and risky.

```ts
export default {
  id: "uplift.config",
  server: async () => ({
    config: async (cfg) => {
      cfg.permission = {
        ...(typeof cfg.permission === "object" ? cfg.permission : {}),
        doom_loop: "ask",
        external_directory: "ask",
      }
    },
  }),
}
```

Use cases:

| Use case | Safer alternative? |
| --- | --- |
| Organization defaults | Managed config or project `opencode.json` is clearer. |
| Plugin self-configuration | Good if documented in plugin README. |
| Runtime provider filtering | Works via `enabled_providers`/`disabled_providers`. |

## Workspace Adapters

Server plugin context includes:

```ts
experimental_workspace: {
  register(type: string, adapter: WorkspaceAdapter): void
}
```

Adapter shape:

```ts
type WorkspaceAdapter = {
  name: string
  description: string
  configure(config: WorkspaceInfo): WorkspaceInfo | Promise<WorkspaceInfo>
  create(config: WorkspaceInfo, env: Record<string, string | undefined>, from?: WorkspaceInfo): Promise<void>
  remove(config: WorkspaceInfo): Promise<void>
  target(config: WorkspaceInfo):
    | { type: "local"; directory: string }
    | { type: "remote"; url: string | URL; headers?: HeadersInit }
    | Promise<{ type: "local"; directory: string } | { type: "remote"; url: string | URL; headers?: HeadersInit }>
}
```

This is a hidden hook point for remote sandboxes, disposable environments, or organization-specific workspace provisioning.

## V2 Event System

`session.next.*` events provide finer telemetry than legacy message/session events. They let tools observe:

| Stage | Events |
| --- | --- |
| Agent/model selection | `session.next.agent.switched`, `session.next.model.switched` |
| Prompt ingestion | `session.next.prompted`, `session.next.synthetic` |
| Shell | `session.next.shell.started`, `session.next.shell.ended` |
| Step lifecycle | `session.next.step.started`, `session.next.step.ended`, `session.next.step.failed` |
| Text stream | `session.next.text.started`, `session.next.text.delta`, `session.next.text.ended` |
| Reasoning stream | `session.next.reasoning.*` |
| Tool lifecycle | `session.next.tool.input.*`, `session.next.tool.called`, `session.next.tool.progress`, `session.next.tool.success`, `session.next.tool.failed` |
| Retry/compaction | `session.next.retried`, `session.next.compaction.*` |

Use these for analytics/timeline visualizers rather than trying to reconstruct everything from `message.part.updated`.

## HTTP API Nuggets

Useful endpoints from official server docs and source:

| Endpoint | Use |
| --- | --- |
| `GET /doc` | OpenAPI 3.1 spec. Use Swagger/codegen. |
| `GET /event` | Instance SSE stream. |
| `GET /global/event` | Global SSE stream across workspaces/instances. |
| `POST /tui/append-prompt` | Append text to active TUI prompt. |
| `POST /tui/execute-command` | Execute TUI command. |
| `POST /tui/show-toast` | Show TUI toast. |
| `GET /experimental/tool/ids` | List tool IDs. |
| `GET /experimental/tool?provider=<p>&model=<m>` | List model-specific tools with JSON schemas. |
| `POST /mcp` | Dynamically add MCP server. |

Protect server with:

```sh
OPENCODE_SERVER_PASSWORD=secret opencode serve
```

Username defaults to `opencode`; override with `OPENCODE_SERVER_USERNAME`.

## Feature Flags Worth Knowing

| Flag | Effect |
| --- | --- |
| `OPENCODE_PURE` | Skip external plugins. |
| `OPENCODE_ENABLE_EXA` | Enable `websearch` outside OpenCode provider. |
| `OPENCODE_ENABLE_QUESTION_TOOL` | Force-enable question tool. |
| `OPENCODE_EXPERIMENTAL_LSP_TOOL` | Enable LSP tool. |
| `OPENCODE_EXPERIMENTAL` | Enables some experimental behavior, including LSP tool according to docs. |
| `OPENCODE_EXPERIMENTAL_EVENT_SYSTEM` | Enables internal TUI Session V2 debug plugin. |

## Compatibility Landmines

| Landmine | Practical Handling |
| --- | --- |
| Types lag runtime | Treat event `type` as string and add default branch. |
| Hook output shapes vary | Shape-check `tool.execute.after`, especially MCP/task. |
| `permission.ask` hook is not called | Use events, permission config, or tool hooks. |
| `session.idle` is deprecated | Prefer `session.status`. |
| Local plugin must export id in new format | Always include stable reverse-DNS style `id`. |
| TUI and server cannot be in same default object | Split files or use package `exports`. |
| Guard via prompt only is weak | Pair `tool.definition` with hard enforcement. |
