# Dialogs And Permission UX

Справочник по пользовательским диалогам в OpenCode: выбор варианта ответа, confirm/prompt/alert, permission approvals и question tool.

Проверено по `packages/plugin/src/tui.ts`, `cli/cmd/tui/plugin/api.tsx`, `ui/dialog-*.tsx`, `routes/session/question.tsx`, `routes/session/permission.tsx`, `question/index.ts`, `permission/index.ts`, SDK V2 generated endpoints.

## Главное Разделение

| Нужно | Механизм | Где работает |
| --- | --- | --- |
| Открыть произвольное окно в TUI | TUI plugin `api.ui.dialog.replace(...)` | Только TUI plugin. |
| Дать пользователю выбрать ответ во время agent execution | Built-in tool `question` | Agent/model tool call. |
| Запросить разрешение на действие | Permission system: config или custom tool `ctx.ask(...)` | Tool execution. |
| Ответить на pending question/permission извне | SDK V2 `client.question.*`, `client.permission.*` | External client/TUI plugin. |
| Показать неблокирующее уведомление | `api.ui.toast(...)` или `/tui/show-toast` | Terminal TUI/API. |

Важное ограничение: server hooks (`tool.execute.before`, `chat.*`, `event`) не имеют прямого API `openDialog()`. Для блокирующего UX из server-side кода используйте permission prompts или question tool. Для произвольных окон нужен TUI plugin.

## TUI Dialog API

TUI plugin получает `api.ui`:

```ts
type DialogStack = {
  replace(render: () => JSX.Element, onClose?: () => void): void
  clear(): void
  setSize(size: "medium" | "large" | "xlarge"): void
  readonly size: "medium" | "large" | "xlarge"
  readonly depth: number
  readonly open: boolean
}
```

Компоненты:

| Component | Props | UX |
| --- | --- | --- |
| `api.ui.DialogAlert` | `{ title, message, onConfirm? }` | Alert с `ok`; Enter подтверждает, Esc закрывает. |
| `api.ui.DialogConfirm` | `{ title, message, onConfirm?, onCancel? }` | Confirm/cancel; left/right переключают кнопку. |
| `api.ui.DialogPrompt` | `{ title, description?, placeholder?, value?, busy?, busyText?, onConfirm?, onCancel? }` | Textarea prompt; Enter submit. |
| `api.ui.DialogSelect` | `{ title, placeholder?, options, flat?, onMove?, onFilter?, onSelect?, skipFilter?, current? }` | Поиск/выбор из списка; Enter выбирает. Сам не закрывается автоматически. |
| `api.ui.Dialog` | `{ size?, onClose, children? }` | Raw wrapper для кастомного JSX. |

Internal `DialogConfirm` implementation also supports `label` for renaming the cancel button, but the public `@opencode-ai/plugin/tui` type at the checked commit does not expose it. Use it only after verifying your installed plugin package types.

`DialogSelect` option:

```ts
type TuiDialogSelectOption<Value = unknown> = {
  title: string
  value: Value
  description?: string
  footer?: JSX.Element | string
  category?: string
  disabled?: boolean
  onSelect?: () => void
}
```

## Loading JSX TUI Plugins

Если plugin использует JSX, удобнее делать directory plugin с `index.tsx` и подключать его явно через config path:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": ["./.opencode/plugins/uplift-tui"]
}
```

Структура:

```text
.opencode/plugins/uplift-tui/index.tsx
```

Авто-скан `.opencode/plugins/*.{ts,js}` не подхватывает `*.tsx` напрямую, но path/directory plugin resolution поддерживает `index.tsx`.

## Example: Select Dialog

```tsx
import type { TuiPlugin } from "@opencode-ai/plugin/tui"

const tui: TuiPlugin = async (api) => {
  function openPolicySelect() {
    api.ui.dialog.replace(() => (
      <api.ui.DialogSelect
        title="Guard mode"
        placeholder="Choose mode"
        current={api.kv.get("guard.mode", "balanced")}
        options={[
          {
            title: "Balanced",
            value: "balanced",
            description: "Block dangerous actions, ask for ambiguous ones.",
            category: "Recommended",
          },
          {
            title: "Strict",
            value: "strict",
            description: "Ask before writes, shell, external directories, and network.",
            category: "Safer",
          },
          {
            title: "Observe",
            value: "observe",
            description: "Only log decisions; do not block.",
            category: "Telemetry",
          },
        ]}
        onSelect={(option) => {
          api.kv.set("guard.mode", option.value)
          api.ui.toast({ variant: "success", message: `Guard mode: ${option.title}` })
          api.ui.dialog.clear()
        }}
      />
    ))
  }

  api.keymap.registerLayer({
    commands: [
      {
        name: "uplift.guard.mode",
        title: "Choose Guard Mode",
        category: "Uplift",
        namespace: "palette",
        run: openPolicySelect,
      },
    ],
  })
}

export default { id: "uplift.dialogs", tui }
```

## Example: Confirm Dialog

```tsx
import type { TuiPluginApi } from "@opencode-ai/plugin/tui"

function confirmCleanup(api: TuiPluginApi) {
  api.ui.dialog.replace(() => (
    <api.ui.DialogConfirm
      title="Run cleanup?"
      message="This will remove generated cache files for the current project."
      onConfirm={() => {
        api.ui.toast({ variant: "info", message: "Cleanup approved" })
      }}
      onCancel={() => {
        api.ui.toast({ variant: "warning", message: "Cleanup cancelled" })
      }}
    />
  ))
}
```

`DialogConfirm` clears the dialog internally after confirm/cancel.

## Example: Prompt Dialog

```tsx
import type { TuiPluginApi } from "@opencode-ai/plugin/tui"

function askForTicket(api: TuiPluginApi) {
  api.ui.dialog.replace(() => (
    <api.ui.DialogPrompt
      title="Attach ticket"
      placeholder="JIRA-123"
      description={() => <text fg={api.theme.current.textMuted}>Optional ticket id for guard logs</text>}
      onConfirm={(value) => {
        api.kv.set("uplift.ticket", value.trim())
        api.ui.toast({ variant: "success", message: "Ticket saved" })
        api.ui.dialog.clear()
      }}
      onCancel={() => api.ui.dialog.clear()}
    />
  ))
}
```

`busy` and `busyText` are useful while an async action is running. When `busy` is true, the prompt suspends input.

## Example: Alert Dialog

```tsx
api.ui.dialog.replace(() => (
  <api.ui.DialogAlert
    title="Guard active"
    message="Dangerous shell commands and secret reads are blocked."
    onConfirm={() => api.ui.toast({ variant: "info", message: "Guard acknowledged" })}
  />
))
```

## Question Tool: Выбор Варианта Во Время Agent Run

OpenCode имеет built-in tool `question`. Это лучший UX, когда именно агенту нужно спросить пользователя и продолжить после ответа.

Tool input:

```ts
type QuestionInfo = {
  question: string
  header: string
  options: Array<{ label: string; description: string }>
  multiple?: boolean
  custom?: boolean
}
```

Поведение TUI:

| Case | UX |
| --- | --- |
| Одна single-choice question | Клик/Enter/цифра сразу отвечает. |
| Несколько questions | Есть tabs по `header` и финальный Confirm. |
| `multiple: true` | Можно выбрать несколько вариантов, затем Confirm. |
| `custom !== false` | Добавляется `Type your own answer`. По умолчанию custom включен. |
| Esc/app exit | `question.rejected`, tool получает `QuestionRejectedError`. |

Пример prompt для агента:

```text
Перед изменением CI конфигурации спроси пользователя через question tool, какой режим выбрать:
1. Fast checks
2. Full matrix
3. Security-only
Рекомендуй Fast checks первым вариантом.
```

Server plugin может усилить инструкцию к `question` tool:

```ts
export default {
  id: "uplift.question-ux",
  server: async () => ({
    "tool.definition": async (input, output) => {
      if (input.toolID !== "question") return
      output.description += "\nUX policy: put the recommended answer first and suffix it with '(Recommended)'."
    },
  }),
}
```

Ограничение: публичный SDK/API умеет list/reply/reject pending questions, но не создает question request напрямую. Создание происходит через built-in `question` tool внутри agent run.

## Reply To Question From SDK V2

External UI или TUI plugin может отвечать на pending questions:

```ts
const pending = await client.question.list()
const first = pending.data?.[0]

if (first) {
  await client.question.reply({
    requestID: first.id,
    answers: [[first.questions[0].options[0].label]],
  })
}
```

Reject:

```ts
await client.question.reject({ requestID: first.id })
```

Events to monitor:

| Event | Meaning |
| --- | --- |
| `question.asked` | New pending question request. |
| `question.replied` | User/client answered. |
| `question.rejected` | User/client dismissed. |

TUI plugin can also read pending questions for a session:

```ts
const pending = api.state.session.question(sessionID)
```

## Permission Prompts: Разрешить Операцию

Permission prompt появляется, когда permission evaluation resolves to `ask`.

Sources:

| Source | How to trigger |
| --- | --- |
| Config | `permission` rules: `bash: "ask"`, `edit: "ask"`, custom keys. |
| Built-in tools | Tools call permission service internally. |
| Custom tools | Call `ctx.ask(...)`. |

Custom tool approval example:

```ts
import { Effect } from "effect"
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Deploy service to staging",
  args: {
    service: tool.schema.string().describe("Service name"),
  },
  async execute(args, ctx) {
    await Effect.runPromise(
      ctx.ask({
        permission: "deploy",
        patterns: [`staging:${args.service}`],
        always: [`staging:${args.service}`],
        metadata: {
          service: args.service,
          environment: "staging",
        },
      }),
    )

    return `Deploy approved for ${args.service}`
  },
})
```

Permission UI options:

| Option | Reply value | Effect |
| --- | --- | --- |
| Allow once | `once` | Approves only this request. |
| Allow always | `always` | Adds `always` patterns to in-memory approvals until OpenCode restart. |
| Reject | `reject` | Fails the tool call. Can include feedback message in some flows. |

TUI has rich permission bodies for known permission keys:

| Permission | UX details |
| --- | --- |
| `edit` | Shows diff from metadata `filepath`/`diff`. |
| `read` | Shows path from tool input `filePath`. |
| `glob`/`grep` | Shows pattern. |
| `bash` | Shows command and description. |
| `task` | Shows subagent type and description. |
| `webfetch` | Shows URL. |
| `websearch` | Shows query. |
| `external_directory` | Shows patterns and derived path. |
| `doom_loop` | Explains repeated failure continuation. |
| custom key | Generic `Call tool <permission>` body. |

For custom permission keys, choose clear `permission`, `patterns`, and `always` strings because the built-in TUI currently shows a generic body. If you need a fully custom approval UI, build a TUI plugin dialog instead.

## Reply To Permission From SDK V2

External UI can list and answer pending permissions:

```ts
const pending = await client.permission.list()
const req = pending.data?.[0]

if (req) {
  await client.permission.reply({
    requestID: req.id,
    reply: "once",
  })
}
```

Reject with feedback:

```ts
await client.permission.reply({
  requestID: req.id,
  reply: "reject",
  message: "Use a read-only dry-run command first.",
})
```

Deprecated older endpoint still exists as SDK `permission.respond({ sessionID, permissionID, response })`; prefer `permission.reply({ requestID, reply })`.

Events to monitor:

| Event | Meaning |
| --- | --- |
| `permission.asked` | New pending permission request. |
| `permission.replied` | Request answered. |

TUI plugin can also read pending permissions for a session:

```ts
const pending = api.state.session.permission(sessionID)
```

## Choosing Between Question, Permission, And TUI Dialog

| Situation | Use | Reason |
| --- | --- | --- |
| Agent needs missing requirement before continuing | `question` tool | Answer is injected back into agent flow. |
| Tool is about to perform risky side effect | `ctx.ask` or `permission` config | Native allow once/always/reject semantics. |
| User is configuring your plugin | TUI `DialogSelect`/`DialogPrompt` | State belongs to plugin, not agent conversation. |
| You need non-blocking status | `api.ui.toast` | Avoids modal friction. |
| External dashboard controls OpenCode | SDK V2 list/reply endpoints | Lets your own UI own the interaction. |

## UX Recommendations

For broader plugin UX, performance and reliability standards, see [`07-plugin-best-practices.md`](07-plugin-best-practices.md).

| Pattern | Recommendation |
| --- | --- |
| Question labels | Keep labels short; put recommended first and append `(Recommended)`. |
| Permission patterns | Make `patterns` and `always` human-readable because users see them. |
| Dialog size | Use `medium` by default, `large/xlarge` for diffs, long lists, or dashboards. |
| Close behavior | `DialogSelect` does not auto-close; call `api.ui.dialog.clear()` after selection. |
| Server hooks | Do not block waiting for arbitrary UI; use permission/question mechanisms. |
| Rejection feedback | Use permission reject `message` to guide the agent toward safer alternatives. |
