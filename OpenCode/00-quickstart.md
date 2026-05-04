# Quickstart: Local Plugin And Tool

End-to-end старт для project-local OpenCode plugin, custom tool и guard verification.

Цель: создать локальный server plugin, добавить custom tool, настроить permissions, запустить OpenCode и проверить, что расширение реально загружено.

## Что Получится

| Surface | Result |
| --- | --- |
| Server plugin | Логирует загрузку, блокирует опасные shell commands, помечает tool output metadata. |
| Custom tool | Добавляет model-callable `repo_policy`. |
| Permission config | Дает безопасные defaults для shell/read/external directories. |
| Verification | Проверка через tool call, blocked command и OpenCode log. |

## Files

```text
your-project/
  opencode.json
  .opencode/
    package.json
    plugins/
      guard.ts
    tools/
      repo_policy.ts
```

## 1. Config

`opencode.json`:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "*": "ask",
    "read": {
      "*": "allow",
      "*.env": "deny",
      "*.env.*": "deny",
      "*.env.example": "allow"
    },
    "bash": {
      "*": "ask",
      "git status*": "allow",
      "git diff*": "allow",
      "git reset --hard*": "deny",
      "git push*": "deny",
      "rm *": "deny"
    },
    "external_directory": "ask",
    "doom_loop": "ask"
  }
}
```

Files in `.opencode/plugins/*.ts` and `.opencode/tools/*.ts` are auto-loaded. Use the `plugin` config field only when you need explicit paths, npm packages, or options.

If you need plugin options, prefer an explicit path outside auto-scanned plugin directories:

```jsonc
{
  "plugin": [["./.opencode/extensions/guard.ts", { "strict": true }]]
}
```

Then move `guard.ts` to `.opencode/extensions/guard.ts` instead of keeping it under `.opencode/plugins/`.

## 2. Dependencies

`.opencode/package.json`:

```json
{
  "type": "module",
  "dependencies": {
    "@opencode-ai/plugin": "latest"
  }
}
```

OpenCode waits for dependencies from the config directory before loading local plugins and tools.

## 3. Server Plugin

`.opencode/plugins/guard.ts`:

```ts
import type { Plugin } from "@opencode-ai/plugin"

const DENY_SHELL = [
  /\brm\s+-rf\s+(\/|\.|~|\*)/,
  /\bgit\s+reset\s+--hard\b/,
  /\bgit\s+push\s+--force\b/,
]

export default {
  id: "uplift.quickstart.guard",
  server: async ({ client, directory }, options) => {
    const strict = options?.strict === true

    await client.app.log({
      body: {
        service: "uplift.quickstart.guard",
        level: "info",
        message: "loaded",
        extra: { directory, strict },
      },
    })

    return {
      "tool.execute.before": async (input, output) => {
        if (input.tool === "bash") {
          const command = String(output.args.command ?? "")
          const blocked = DENY_SHELL.find((rule) => rule.test(command))
          if (blocked) throw new Error(`Blocked dangerous shell command: ${command}`)
        }

        if (input.tool === "read") {
          const file = String(output.args.filePath ?? "")
          if (/\.env(?:\.|$)/.test(file) && !file.endsWith(".env.example")) {
            throw new Error(`Blocked secret file read: ${file}`)
          }
        }
      },

      "tool.execute.after": async (_input, output: any) => {
        if (!output || typeof output !== "object" || !("metadata" in output)) return
        output.metadata = { ...(output.metadata ?? {}), quickstartGuard: true }
      },
    }
  },
} satisfies { id: string; server: Plugin }
```

Use `tool.execute.before` for enforcement. `tool.definition` can influence model behavior, but it is not a security boundary.

## 4. Custom Tool

`.opencode/tools/repo_policy.ts`:

```ts
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Return the repository guard policy summary",
  args: {},
  async execute(_args, ctx) {
    ctx.metadata({ title: "Repository policy" })

    return {
      output: [
        `Directory: ${ctx.directory}`,
        `Worktree: ${ctx.worktree}`,
        "Policy: dangerous shell commands and secret reads are blocked.",
      ].join("\n"),
      metadata: { source: "repo_policy" },
    }
  },
})
```

File name becomes the tool id, so this tool is exposed as `repo_policy`.

## 5. Run

From the project root:

```sh
opencode
```

For headless/API verification:

```sh
opencode serve --port 4096
```

Do not set `OPENCODE_PURE=1` while testing external plugins; pure mode skips them.

## 6. Verify

| Check | How | Expected |
| --- | --- | --- |
| Plugin loaded | Inspect OpenCode logs for service `uplift.quickstart.guard`. | Log entry with message `loaded`. |
| Custom tool exists | Ask: `Use the repo_policy tool and print its result.` | Assistant calls `repo_policy` and shows policy text. |
| Safe shell still works | Ask: `Run git status --short.` | Command runs automatically or asks, depending on your merged config. |
| Dangerous shell is blocked | Ask: `Run git reset --hard.` in a disposable test repo. | Tool call is denied by permission config or blocked by plugin before execution. |
| Secret read is blocked | Ask to read `.env` in a disposable test repo. | `read` is denied by default config or blocked by plugin. |

For API-driven checks, inspect tool ids:

```sh
curl http://localhost:4096/experimental/tool/ids
```

`repo_policy` should appear when the server is running with this project config.

## 7. Package As NPM Plugin

Minimal server-only package shape:

```text
opencode-uplift-guard/
  package.json
  src/server.ts
  dist/server.js
```

`package.json`:

```json
{
  "name": "opencode-uplift-guard",
  "type": "module",
  "exports": {
    "./server": "./dist/server.js"
  },
  "engines": {
    "opencode": ">=1.0.0 <2"
  },
  "dependencies": {
    "@opencode-ai/plugin": "latest"
  }
}
```

`src/server.ts` can export the same default object as the local plugin.

Consumers add it to `opencode.json`:

```jsonc
{
  "plugin": ["opencode-uplift-guard"]
}
```

If the same package also ships a TUI plugin, split entrypoints with `exports["./server"]` and `exports["./tui"]`. A single default plugin object cannot contain both `server` and `tui`.

## Next Reading

| Need | Read |
| --- | --- |
| Runtime, load order, options | [`01-plugin-runtime.md`](01-plugin-runtime.md) |
| Full hook signatures | [`02-hooks-reference.md`](02-hooks-reference.md) |
| Event bus and SSE | [`03-events-reference.md`](03-events-reference.md) |
| Guard recipes | [`04-custom-tools-and-guards.md`](04-custom-tools-and-guards.md) |
| TUI dialogs and permission UX | [`06-dialogs-and-permissions.md`](06-dialogs-and-permissions.md) |
| Production UX/performance standards | [`07-plugin-best-practices.md`](07-plugin-best-practices.md) |
