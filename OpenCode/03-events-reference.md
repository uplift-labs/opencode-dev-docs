# Events Reference

Справочник OpenCode event bus/SSE events для plugins, SDK clients и внешних инструментов.

Проверено по `packages/opencode/src/bus/*`, `server/routes/*/event.ts`, `packages/sdk/js/src/v2/gen/types.gen.ts` и event definitions в `file`, `session`, `permission`, `question`, `mcp`, `workspace`, `worktree`, `tui`, `pty`.

## Envelopes

Instance event stream `/event` and server plugin `event` hook use:

```ts
type BusEvent = {
  id: string
  type: string
  properties: Record<string, unknown>
}
```

Global event stream `/global/event` wraps payload:

```ts
type GlobalEvent = {
  directory?: string
  project?: string
  workspace?: string
  payload: BusEvent | SyncEvent
}
```

Transport notes:

| Event | Behavior |
| --- | --- |
| `server.connected` | First event sent when SSE connection opens. |
| `server.heartbeat` | Synthetic heartbeat every 10 seconds; not part of older generated SDK event unions. |
| `server.instance.disposed` | Published before instance bus shuts down; instance stream stops after it. |
| `global.disposed` | Global lifecycle event after all instances are disposed. |

## Subscribe Examples

### Server Plugin

```ts
export default {
  id: "uplift.events",
  server: async ({ client }) => ({
    event: async ({ event }) => {
      if (event.type === "server.heartbeat") return
      await client.app.log({
        body: { service: "uplift.events", level: "debug", message: event.type, extra: event.properties },
      })
    },
  }),
}
```

Do not confuse hooks with events: `tool.execute.after` is a blocking hook, not a bus event.

### SDK Client

```ts
import { createOpencodeClient } from "@opencode-ai/sdk"

const client = createOpencodeClient({ baseUrl: "http://localhost:4096" })
const events = await client.event.subscribe()

for await (const event of events.stream) {
  if (event.type === "session.status") {
    console.log(event.properties.sessionID, event.properties.status.type)
  }
}
```

### Raw SSE

```ts
const res = await fetch("http://localhost:4096/event")
// Each SSE message data is JSON BusEvent.
```

## Core Transport Events

| Type | Properties | Meaning |
| --- | --- | --- |
| `server.connected` | `{}` | SSE connection opened. |
| `server.heartbeat` | `{}` | Keepalive every 10s. Hidden synthetic event. |
| `server.instance.disposed` | `{ directory: string }` | Instance is disposing. |
| `global.disposed` | `{}` | Global server lifecycle disposed all instances. |

## Installation Events

| Type | Properties | Meaning |
| --- | --- | --- |
| `installation.updated` | `{ version: string }` | OpenCode updated to a new version. |
| `installation.update-available` | `{ version: string }` | New update is available. |

## File, LSP, MCP Events

| Type | Properties | Meaning |
| --- | --- | --- |
| `file.edited` | `{ file: string }` | A file was modified through OpenCode file service/tooling. |
| `file.watcher.updated` | `{ file: string, event: "add" | "change" | "unlink" }` | Filesystem watcher observed change. |
| `lsp.client.diagnostics` | `{ serverID: string, path: string }` | LSP diagnostics changed for a path. |
| `lsp.updated` | `{}` | LSP server status/list changed. |
| `mcp.tools.changed` | `{ server: string }` | MCP server tool list changed. Hidden in plugin docs. |
| `mcp.browser.open.failed` | `{ mcpName: string, url: string }` | MCP OAuth/browser launch failed. Hidden in plugin docs. |

## Project, VCS, Workspace Events

| Type | Properties | Meaning |
| --- | --- | --- |
| `project.updated` | `Project` | Project metadata changed. Hidden in plugin docs. |
| `vcs.branch.updated` | `{ branch?: string }` | Current VCS branch changed. Hidden in plugin docs. |
| `workspace.ready` | `{ name: string }` | Control-plane workspace is ready. Hidden/experimental. |
| `workspace.failed` | `{ message: string }` | Workspace creation/connection failed. Hidden/experimental. |
| `workspace.restore` | `{ workspaceID: string, sessionID: string, total: number, step: number }` | Workspace restore progress. Hidden/experimental. |
| `workspace.status` | `{ workspaceID: string, status: "connected" | "connecting" | "disconnected" | "error" }` | Workspace connection status. Hidden/experimental. |
| `worktree.ready` | `{ name: string, branch: string }` | Git worktree creation ready. Hidden in plugin docs. |
| `worktree.failed` | `{ message: string }` | Git worktree creation failed. Hidden in plugin docs. |

## Session Events

| Type | Properties | Meaning |
| --- | --- | --- |
| `session.created` | `{ sessionID: string, info: Session }` | Session created. |
| `session.updated` | `{ sessionID: string, info: Session }` | Session changed. For sync storage, `info` can be a patch; bus schema publishes full info. |
| `session.deleted` | `{ sessionID: string, info: Session }` | Session deleted. |
| `session.diff` | `{ sessionID: string, diff: SnapshotFileDiff[] }` | Session file diff changed. |
| `session.error` | `{ sessionID?: string, error?: ErrorUnion }` | Session-level error. |
| `session.status` | `{ sessionID: string, status: SessionStatus }` | Session is `idle`, `busy`, or `retry`. |
| `session.idle` | `{ sessionID: string }` | Deprecated compatibility event. Prefer `session.status`. |
| `session.compacted` | `{ sessionID: string }` | Session compaction completed. |

`SessionStatus`:

```ts
type SessionStatus =
  | { type: "idle" }
  | { type: "busy" }
  | { type: "retry"; attempt: number; message: string; next: number }
```

## Message Events

| Type | Properties | Meaning |
| --- | --- | --- |
| `message.updated` | `{ sessionID: string, info: Message }` | User or assistant message was created/updated. |
| `message.removed` | `{ sessionID: string, messageID: string }` | Message removed. |
| `message.part.updated` | `{ sessionID: string, part: Part, time: number }` | Message part created/updated. |
| `message.part.delta` | `{ sessionID: string, messageID: string, partID: string, field: string, delta: string }` | Streaming delta for a part field, usually assistant text. Hidden in plugin docs. |
| `message.part.removed` | `{ sessionID: string, messageID: string, partID: string }` | Message part removed. |

Important `Part.type` values from SDK V2:

| Part type | Meaning |
| --- | --- |
| `text` | Text content, can be synthetic or ignored. |
| `subtask` | User/requested subagent invocation part. |
| `reasoning` | Provider reasoning text. |
| `file` | File/media attachment. |
| `tool` | Tool call state. |
| `step-start` | Agentic step started. |
| `step-finish` | Agentic step ended with tokens/cost. |
| `snapshot` | Snapshot marker. |
| `patch` | Patch summary part. |
| `agent` | Agent mention/attachment. |
| `retry` | Provider retry info. |
| `compaction` | Compaction marker. |

## Permission And Question Events

| Type | Properties | Meaning |
| --- | --- | --- |
| `permission.asked` | `PermissionRequest` | OpenCode asks user to approve/deny an action. |
| `permission.replied` | `{ sessionID: string, requestID: string, reply: "once" | "always" | "reject" }` | User answered permission prompt. |
| `question.asked` | `QuestionRequest` | Question tool asks user. Hidden in plugin docs. |
| `question.replied` | `{ sessionID: string, requestID: string, answers: string[][] }` | User answered questions. Hidden in plugin docs. |
| `question.rejected` | `{ sessionID: string, requestID: string }` | User dismissed question. Hidden in plugin docs. |

`PermissionRequest`:

```ts
type PermissionRequest = {
  id: string
  sessionID: string
  permission: string
  patterns: string[]
  metadata: Record<string, unknown>
  always: string[]
  tool?: { messageID: string; callID: string }
}
```

## Todo And Command Events

| Type | Properties | Meaning |
| --- | --- | --- |
| `todo.updated` | `{ sessionID: string, todos: Todo[] }` | Todo list replaced/updated. |
| `command.executed` | `{ name: string, sessionID: string, arguments: string, messageID: string }` | Slash/custom command executed. |

`Todo` currently has `content`, `status`, `priority`; older SDK v1 also had `id`, but source V2 `Todo.Info` does not.

## TUI Control Events

These events are both bus events and HTTP/SDK TUI control primitives.

| Type | Properties | Meaning |
| --- | --- | --- |
| `tui.prompt.append` | `{ text: string }` | Append text to current TUI prompt. |
| `tui.command.execute` | `{ command: string }` | Execute TUI command. Known commands include `session.list`, `session.new`, `session.share`, `session.interrupt`, `session.compact`, `session.page.up`, `session.page.down`, `session.line.up`, `session.line.down`, `session.half.page.up`, `session.half.page.down`, `session.first`, `session.last`, `prompt.clear`, `prompt.submit`, `agent.cycle`. |
| `tui.toast.show` | `{ title?: string, message: string, variant: "info" | "success" | "warning" | "error", duration?: number }` | Show toast. Default duration is 5000ms. |
| `tui.session.select` | `{ sessionID: string }` | Navigate/select session. Hidden in plugin docs. |

## PTY Events

| Type | Properties | Meaning |
| --- | --- | --- |
| `pty.created` | `{ info: Pty }` | Interactive terminal session created. Hidden in plugin docs. |
| `pty.updated` | `{ info: Pty }` | PTY metadata/status changed. Hidden in plugin docs. |
| `pty.exited` | `{ id: string, exitCode: number }` | PTY process exited. Hidden in plugin docs. |
| `pty.deleted` | `{ id: string }` | PTY session deleted. Hidden in plugin docs. |

`Pty`:

```ts
type Pty = {
  id: string
  title: string
  command: string
  args: string[]
  cwd: string
  status: "running" | "exited"
  pid: number
}
```

## V2 Session Event System: `session.next.*`

These are high-signal lifecycle events for the newer session engine. They are generated from `packages/opencode/src/v2/session-event.ts` and appear in SDK V2 types.

Common fields: most have `{ timestamp: number, sessionID: string }`.

| Type | Extra properties | Meaning |
| --- | --- | --- |
| `session.next.agent.switched` | `{ agent: string }` | Active agent changed. |
| `session.next.model.switched` | `{ id: string, providerID: string, variant?: string }` | Active model changed. |
| `session.next.prompted` | `{ prompt: Prompt }` | User prompt accepted by V2 flow. |
| `session.next.synthetic` | `{ text: string }` | Synthetic user/system text inserted. |
| `session.next.shell.started` | `{ callID: string, command: string }` | Shell command started. |
| `session.next.shell.ended` | `{ callID: string, output: string }` | Shell command ended. |
| `session.next.step.started` | `{ agent: string, model: { id, providerID, variant? }, snapshot?: string }` | Agent step started. |
| `session.next.step.ended` | `{ finish: string, cost: number, tokens, snapshot?: string }` | Agent step ended. |
| `session.next.step.failed` | `{ error: { type: string, message: string } }` | Agent step failed. |
| `session.next.text.started` | `{}` | Assistant text part started. |
| `session.next.text.delta` | `{ delta: string }` | Text stream delta. |
| `session.next.text.ended` | `{ text: string }` | Text stream complete. |
| `session.next.reasoning.started` | `{ reasoningID: string }` | Reasoning stream started. |
| `session.next.reasoning.delta` | `{ reasoningID: string, delta: string }` | Reasoning delta. |
| `session.next.reasoning.ended` | `{ reasoningID: string, text: string }` | Reasoning complete. |
| `session.next.tool.input.started` | `{ callID: string, name: string }` | Tool input streaming started. |
| `session.next.tool.input.delta` | `{ callID: string, delta: string }` | Tool input JSON/text delta. |
| `session.next.tool.input.ended` | `{ callID: string, text: string }` | Tool input complete. |
| `session.next.tool.called` | `{ callID: string, tool: string, input: object, provider: { executed: boolean, metadata?: object } }` | Tool call emitted. |
| `session.next.tool.progress` | `{ callID: string, structured: object, content: ToolContent[] }` | Tool progress event. |
| `session.next.tool.success` | `{ callID: string, structured: object, content: ToolContent[], provider }` | Tool succeeded. |
| `session.next.tool.failed` | `{ callID: string, error: { type, message }, provider }` | Tool failed. |
| `session.next.retried` | `{ attempt: number, error: SessionNextRetryError }` | Provider retry happened. |
| `session.next.compaction.started` | `{ reason: "auto" | "manual" }` | V2 compaction started. |
| `session.next.compaction.delta` | `{ text: string }` | Compaction text delta. |
| `session.next.compaction.ended` | `{ text: string, include?: string }` | Compaction complete. |

`Prompt`:

```ts
type Prompt = {
  text: string
  files?: Array<{ uri: string; mime: string; name?: string; description?: string; source?: PromptSource }>
  agents?: Array<{ name: string; source?: PromptSource }>
}
```

## Sync Events

SDK V2 also exposes sync envelopes for persisted/replayable session events:

```ts
type SyncEvent = {
  type: "sync"
  name: "message.updated.1" | "session.created.1" | "session.next.text.delta.1" | string
  id: string
  seq: number
  aggregateID: "sessionID"
  data: object
}
```

Sync events mirror these bus event families:

| Sync name family | Mirrors |
| --- | --- |
| `message.updated.1`, `message.removed.1`, `message.part.updated.1`, `message.part.removed.1` | Message events. |
| `session.created.1`, `session.updated.1`, `session.deleted.1` | Session events. |
| `session.next.*.1` | V2 session lifecycle events. |

For plugin guards, prefer normal bus events and hooks. For external clients building replay/sync features, keep sync events in your parser.
