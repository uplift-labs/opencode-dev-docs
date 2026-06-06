# Custom Tools And Guards

Практические паттерны для hooks-focused разработки OpenCode инструментов.

Проверено по `packages/plugin/src/tool.ts`, `packages/opencode/src/tool/registry.ts`, `tool/shell/id.ts`, official `custom-tools`, `tools`, `permissions` docs.

## Custom Tool Locations

| Scope | Path |
| --- | --- |
| Project | `.opencode/tools/*.ts` or `.opencode/tools/*.js` |
| Global | `~/.config/opencode/tools/*.ts` or `*.js` |
| Compatibility | Source scans both `tool/` and `tools/`. |

Default export naming:

```ts
// .opencode/tools/database.ts -> tool id: database
export default tool({ ... })
```

Named exports:

```ts
// .opencode/tools/math.ts -> tool ids: math_add, math_multiply
export const add = tool({ ... })
export const multiply = tool({ ... })
```

Server plugin tool naming:

```ts
return {
  tool: {
    my_tool: tool({ ... }),
  },
}
```

## Tool Definition

```ts
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Query an internal API",
  args: {
    query: tool.schema.string().describe("Query text"),
  },
  async execute(args, context) {
    return `received: ${args.query}`
  },
})
```

`tool.schema` is Zod.

## Tool Context

| Field | Meaning |
| --- | --- |
| `sessionID` | Current session. |
| `messageID` | Current assistant/user message context. |
| `agent` | Current agent name. |
| `directory` | Current project/session directory. |
| `worktree` | Git worktree root. |
| `abort` | AbortSignal for cancellation. |
| `metadata({ title?, metadata? })` | Update running tool part metadata/title. |
| `ask(input)` | Ask permission through OpenCode permission system. Returns Effect. |

Return type:

```ts
type ToolResult = string | { output: string; metadata?: Record<string, any> }
```

OpenCode truncates custom tool output according to agent/tool output policy and records truncation metadata.

## Permission-Aware Tool

```ts
import { Effect } from "effect"
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Deploy to staging",
  args: {
    service: tool.schema.string(),
  },
  async execute(args, ctx) {
    await Effect.runPromise(
      ctx.ask({
        permission: "deploy",
        patterns: [`staging:${args.service}`],
        always: [`staging:${args.service}`],
        metadata: { service: args.service },
      }),
    )

    return `Deployment approved for ${args.service}`
  },
})
```

Custom permission names can be governed by config because permission config accepts additional keys.

## Guard: Block Env File Reads

```ts
export default {
  id: "uplift.guard.env",
  server: async () => ({
    "tool.execute.before": async (input, output) => {
      if (input.tool !== "read") return
      const file = String(output.args.filePath ?? output.args.path ?? "")
      if (/\.env(\.|$)/.test(file) && !file.endsWith(".env.example")) {
        throw new Error(`Blocked secret file read: ${file}`)
      }
    },
  }),
}
```

OpenCode default permissions already deny `.env` reads. Keep this pattern when you need stronger organization-specific messages or telemetry.

## Guard: Block Dangerous Shell

```ts
const DENY = [
  /\brm\s+-rf\s+(\/|\.|~|\*)/,
  /\bgit\s+push\s+--force\b/,
  /\bgit\s+reset\s+--hard\b/,
]

export default {
  id: "uplift.guard.shell",
  server: async ({ client }) => ({
    "tool.execute.before": async (input, output) => {
      if (input.tool !== "bash") return
      const command = String(output.args.command ?? "")
      const hit = DENY.find((rule) => rule.test(command))
      if (!hit) return

      await client.app.log({
        body: {
          service: "uplift.guard.shell",
          level: "warn",
          message: "blocked shell command",
          extra: { sessionID: input.sessionID, callID: input.callID, command },
        },
      })

      throw new Error(`Blocked dangerous shell command: ${command}`)
    },
  }),
}
```

## Guard: Patch Path Policy

`apply_patch` uses `output.args.patchText`, not `filePath`.

```ts
function patchPaths(patchText: string) {
  return [...patchText.matchAll(/^\*\*\* (?:Add|Update|Delete) File: (.+)$/gm)].map((m) => m[1])
}

export default {
  id: "uplift.guard.patch",
  server: async () => ({
    "tool.execute.before": async (input, output) => {
      if (input.tool !== "apply_patch") return
      const paths = patchPaths(String(output.args.patchText ?? ""))
      const blocked = paths.find((p) => p.startsWith("secrets/") || p.includes(".env"))
      if (blocked) throw new Error(`Blocked patch touching protected path: ${blocked}`)
    },
  }),
}
```

## Guard: Add Context To Tool Definitions

```ts
export default {
  id: "uplift.tool-policy-prompt",
  server: async () => ({
    "tool.definition": async (input, output) => {
      if (input.toolID === "bash") {
        output.description += "\nPolicy: prefer read-only commands first; ask user before destructive operations."
      }
      if (input.toolID === "read") {
        output.description += "\nPolicy: do not read secrets, credentials, or private keys."
      }
    },
  }),
}
```

This improves model behavior but does not enforce policy. Pair it with `permission` or `tool.execute.before`.

## Guard: Post-Process Tool Output

```ts
export default {
  id: "uplift.output-redactor",
  server: async () => ({
    "tool.execute.after": async (input, output: any) => {
      if (!output || typeof output.output !== "string") return
      output.output = output.output.replace(/sk-[A-Za-z0-9_-]{20,}/g, "[REDACTED_API_KEY]")
      output.metadata = { ...(output.metadata ?? {}), redacted: true }
    },
  }),
}
```

For MCP tools, `output` can be raw MCP result. Always shape-check.

## Shell Env Injection

```ts
export default {
  id: "uplift.env",
  server: async () => ({
    "shell.env": async (input, output) => {
      output.env.UPLIFT_SESSION = input.sessionID ?? "pty"
      output.env.UPLIFT_CWD = input.cwd
    },
  }),
}
```

This affects `bash` tool, session shell command, and PTY terminal. For PTY, `sessionID` and `callID` can be absent.

## Permissions Config Pairing

Use config for broad defaults and hooks for semantic policy.

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
      "git push*": "deny",
      "rm *": "deny"
    },
    "external_directory": "ask",
    "doom_loop": "ask"
  }
}
```

Available built-in permission keys include `read`, `edit`, `glob`, `grep`, `bash`, `task`, `external_directory`, `todowrite`, `question`, `webfetch`, `websearch`, `lsp`, `doom_loop`, `skill`, plus custom tool/permission names. Directory listing is handled by the `read` tool/permission; there is no separate built-in `list` permission in the checked source.

## Tool ID Caveats

| Topic | Detail |
| --- | --- |
| Shell rename | Source renamed `bash.ts` to `shell.ts`, but exposed tool ID and permission key remain `bash` until OpenCode 2.0. |
| GPT patch mode | Registry may expose `apply_patch` instead of `edit`/`write` for some GPT model IDs. Guard both edit family and `apply_patch`. |
| Question tool | Enabled for app/cli/desktop clients or `OPENCODE_ENABLE_QUESTION_TOOL`. |
| Websearch | Enabled for OpenCode provider or truthy `OPENCODE_ENABLE_EXA`. |
| LSP tool | Requires `OPENCODE_EXPERIMENTAL_LSP_TOOL=true` or experimental flag. |
