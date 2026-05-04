# Hooks Reference

Swagger-like справочник server-side hooks из `@opencode-ai/plugin`.

Проверено по `packages/plugin/src/index.ts` и местам вызова в `session/llm.ts`, `session/prompt.ts`, `session/processor.ts`, `session/compaction.ts`, `tool/registry.ts`, `tool/shell.ts`, `provider/auth.ts`, `provider/provider.ts`.

## Contract

Большинство hooks имеют форму:

```ts
"hook.name": async (input, output) => {
  // mutate output in-place
}
```

Правила:

| Rule | Detail |
| --- | --- |
| Mutate output | OpenCode использует тот же object после hook chain. |
| Sequential | Hooks выполняются по порядку загрузки plugins. |
| Throw to block | Ошибка в trigger hook обычно прерывает исходное действие. |
| Event is special | `event` fire-and-forget, не await, не guard. |
| Types can lag runtime | Runtime может прислать V2 events, даже если тип `Event` из `@opencode-ai/sdk` старее. |

## Lifecycle And Object Hooks

### `event`

Слушает все bus events текущего instance.

```ts
event?: (input: { event: Event }) => Promise<void>
```

Payload envelope:

```ts
{
  id: string
  type: string
  properties: object
}
```

Example:

```ts
export default {
  id: "uplift.event-log",
  server: async ({ client }) => ({
    event: async ({ event }) => {
      if (event.type === "session.error") {
        await client.app.log({
          body: { service: "uplift.event-log", level: "error", message: "session error", extra: event.properties },
        })
      }
    },
  }),
}
```

Use for telemetry, notifications, dashboards. Do not use for blocking tools.

### `config`

Called once after plugins load and before provider state is built. Receives mutable resolved config.

```ts
config?: (input: Config) => Promise<void>
```

Example:

```ts
config: async (cfg) => {
  cfg.disabled_providers = [...(cfg.disabled_providers ?? []), "gemini"]
}
```

Errors are logged and ignored per plugin.

### `tool`

Registers custom tools from a server plugin.

```ts
tool?: Record<string, ToolDefinition>
```

Example:

```ts
import { tool } from "@opencode-ai/plugin"

tool: {
  repo_policy: tool({
    description: "Read repository policy summary",
    args: {},
    async execute(args, ctx) {
      return `Session ${ctx.sessionID} in ${ctx.worktree}`
    },
  }),
}
```

Plugin tools are appended after built-ins. If a custom tool reuses a built-in name, it can take precedence in the final tool map.

### `auth`

Registers provider auth methods and optional credential loader.

```ts
auth?: {
  provider: string
  loader?: (auth: () => Promise<Auth>, provider: Provider) => Promise<Record<string, any>>
  methods: Array<ApiMethod | OAuthMethod>
}
```

High-value use cases:

| Use case | Hook field |
| --- | --- |
| Override built-in provider auth | `auth.provider` with same provider id. |
| Add API-key prompt | `methods: [{ type: "api", label, prompts, authorize }]`. |
| Add OAuth flow | `methods: [{ type: "oauth", label, authorize }]`. |
| Transform stored auth into provider vars | `loader`. |

Prompt rule fields support `when: { key, op: "eq" | "neq", value }`. Older `condition()` is deprecated in types.

### `provider`

Extends model discovery for an existing provider.

```ts
provider?: {
  id: string
  models?: (provider: ProviderV2, ctx: { auth?: Auth }) => Promise<Record<string, ModelV2>>
}
```

Source behavior: plugin provider model hooks only run when the provider exists in the models database and is not disabled by config.

## Chat Hooks

### `chat.message`

Called after user prompt parts are resolved and before the user message/parts are saved.

```ts
"chat.message"?: (
  input: {
    sessionID: string
    agent?: string
    model?: { providerID: string; modelID: string }
    messageID?: string
    variant?: string
  },
  output: { message: UserMessage; parts: Part[] },
) => Promise<void>
```

Use cases:

| Use case | Action |
| --- | --- |
| Prompt enrichment | Push synthetic text/file parts. |
| Prompt policy | Reject or rewrite user parts. |
| Agent routing metadata | Read `agent`, `model`, `variant`. |

Example:

```ts
"chat.message": async (input, output) => {
  if (input.agent === "build") {
    output.message.system = [output.message.system, "Always report changed files at the end."].filter(Boolean).join("\n")
  }
}
```

### `chat.params`

Called immediately before `streamText()` provider call.

```ts
"chat.params"?: (
  input: { sessionID: string; agent: string; model: Model; provider: ProviderContext; message: UserMessage },
  output: {
    temperature: number | undefined
    topP: number
    topK: number
    maxOutputTokens: number | undefined
    options: Record<string, any>
  },
) => Promise<void>
```

Example:

```ts
"chat.params": async ({ agent, provider }, output) => {
  if (agent === "code-reviewer") output.temperature = 0
  if (provider.info.id === "anthropic") output.options.setCacheKey = true
}
```

### `chat.headers`

Adds provider request headers.

```ts
"chat.headers"?: (
  input: { sessionID: string; agent: string; model: Model; provider: ProviderContext; message: UserMessage },
  output: { headers: Record<string, string> },
) => Promise<void>
```

Example:

```ts
"chat.headers": async (input, output) => {
  output.headers["x-uplift-session"] = input.sessionID
}
```

## Command Hooks

### `command.execute.before`

Called before slash/custom command template is sent as a prompt.

```ts
"command.execute.before"?: (
  input: { command: string; sessionID: string; arguments: string },
  output: { parts: Part[] },
) => Promise<void>
```

Example:

```ts
"command.execute.before": async (input, output) => {
  if (input.command === "review") {
    output.parts.push({ type: "text", text: "Also check for secret leakage." } as any)
  }
}
```

## Tool Hooks

### `tool.execute.before`

Called before built-in tools, custom plugin tools, MCP tools and task/subagent tool execute.

```ts
"tool.execute.before"?: (
  input: { tool: string; sessionID: string; callID: string },
  output: { args: any },
) => Promise<void>
```

Canonical guard example:

```ts
"tool.execute.before": async (input, output) => {
  if (input.tool === "bash" && /\brm\s+-rf\b/.test(output.args.command ?? "")) {
    throw new Error("Blocked destructive shell command")
  }
}
```

Mutation example:

```ts
"tool.execute.before": async (input, output) => {
  if (input.tool === "bash") {
    output.args.command = `set -e; ${output.args.command}`
  }
}
```

Important tool IDs:

| Tool ID | Note |
| --- | --- |
| `bash` | Source file is now `shell.ts`, but exposed id and permission key remain `bash` for compatibility. |
| `apply_patch` | Args use `patchText`; paths are embedded in patch markers. |
| `task` | Subagent tool. Args include `prompt`, `description`, `subagent_type`, `command`. |
| MCP tools | IDs usually come from MCP server/tool naming; output shapes differ in `after`. |

### `tool.execute.after`

Called after tool execution and before OpenCode persists/display-converts result.

```ts
"tool.execute.after"?: (
  input: { tool: string; sessionID: string; callID: string; args: any },
  output: { title: string; output: string; metadata: any },
) => Promise<void>
```

Runtime caveat from source: MCP tools pass the raw MCP result to this hook before OpenCode converts it to `{ title, output, metadata }`. Task tool can pass `undefined` on failure. Defensive code should shape-check.

```ts
"tool.execute.after": async (input, output: any) => {
  if (!output) return
  if ("output" in output && typeof output.output === "string") {
    output.metadata = { ...(output.metadata ?? {}), audited: true }
  }
}
```

### `tool.definition`

Called while building tool definitions for the LLM. Mutates description and JSON/Zod-derived parameter schema before the model sees it.

```ts
"tool.definition"?: (
  input: { toolID: string },
  output: { description: string; parameters: any },
) => Promise<void>
```

Example:

```ts
"tool.definition": async (input, output) => {
  if (input.toolID === "bash") {
    output.description += "\nRepository policy: never run destructive git or filesystem commands without explicit user request."
  }
}
```

Do not use this as the only security layer. It influences the model, but enforcement belongs in permissions or `tool.execute.before`.

## Shell Hooks

### `shell.env`

Called before shell execution in three paths: bash tool, session shell command, and PTY terminal creation.

```ts
"shell.env"?: (
  input: { cwd: string; sessionID?: string; callID?: string },
  output: { env: Record<string, string> },
) => Promise<void>
```

Example:

```ts
"shell.env": async (input, output) => {
  output.env.UPLIFT_PROJECT_ROOT = input.cwd
  output.env.NO_COLOR = "1"
}
```

OpenCode merges `process.env` with `output.env`. PTY calls may omit `sessionID` and `callID`.

## Experimental Chat/System Hooks

### `experimental.chat.messages.transform`

Called before message history is converted to provider model messages. Used in normal loop and compaction.

```ts
"experimental.chat.messages.transform"?: (
  input: {},
  output: { messages: { info: Message; parts: Part[] }[] },
) => Promise<void>
```

Use carefully. Removing tool-call context can break provider validation. Prefer marking content as synthetic/ignored over destructive rewrites when possible.

### `experimental.chat.system.transform`

Called before LLM calls after system prompt assembly. Also used during generated agent creation where `sessionID` can be absent.

```ts
"experimental.chat.system.transform"?: (
  input: { sessionID?: string; model: Model },
  output: { system: string[] },
) => Promise<void>
```

Example:

```ts
"experimental.chat.system.transform": async (input, output) => {
  output.system.push("When writing files, prefer minimal diffs and explain verification evidence.")
}
```

### `experimental.text.complete`

Called when an assistant text part ends, before final part update.

```ts
"experimental.text.complete"?: (
  input: { sessionID: string; messageID: string; partID: string },
  output: { text: string },
) => Promise<void>
```

Example:

```ts
"experimental.text.complete": async (input, output) => {
  output.text = output.text.replace(/\s+$/g, "")
}
```

## Compaction Hooks

### `experimental.session.compacting`

Called before OpenCode generates a continuation summary.

```ts
"experimental.session.compacting"?: (
  input: { sessionID: string },
  output: { context: string[]; prompt?: string },
) => Promise<void>
```

Behavior:

| Field | Effect |
| --- | --- |
| `output.context.push(text)` | Appends context to default compaction prompt. |
| `output.prompt = text` | Replaces the whole default compaction prompt. `context` is ignored. |

### `experimental.compaction.autocontinue`

Called after successful compaction and before OpenCode adds synthetic user `continue`.

```ts
"experimental.compaction.autocontinue"?: (
  input: {
    sessionID: string
    agent: string
    model: Model
    provider: ProviderContext
    message: UserMessage
    overflow: boolean
  },
  output: { enabled: boolean },
) => Promise<void>
```

Example:

```ts
"experimental.compaction.autocontinue": async (input, output) => {
  if (!input.overflow) output.enabled = false
}
```

## Declared But Currently Not Triggered

### `permission.ask`

The type exists in `packages/plugin/src/index.ts`:

```ts
"permission.ask"?: (input: Permission, output: { status: "ask" | "deny" | "allow" }) => Promise<void>
```

Current source does not call `plugin.trigger("permission.ask", ...)`. Use `permission` config, `tool.execute.before`, or listen to `permission.asked`/`permission.replied` events instead.
