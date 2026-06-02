# OpenCode Tooling Docs

Эта папка собрана как практический справочник для разработки OpenCode hooks, plugins, custom tools и интеграций.

Источник данных:

- Официальная документация: https://opencode.ai/docs, страницы `plugins`, `config`, `sdk`, `server`, `tools`, `custom-tools`, `permissions`.
- Исходники: `anomalyco/opencode`, ветка `dev`; базовые hook notes сверялись с `9f708e748af34cf63c0b1010c4a07ddab1b10ef6`, Desktop/TUI deltas сверены с `18ba80f` на `2026-06-02`.
- Ключевые файлы исходников: `packages/plugin/src/index.ts`, `packages/plugin/src/tui.ts`, `packages/opencode/src/plugin/*`, `packages/opencode/src/bus/*`, `packages/opencode/src/session/*`, `packages/opencode/src/tool/*`, `packages/app/src/*`, `packages/desktop/src/*`, `packages/sdk/js/src/v2/gen/types.gen.ts`.

## Документы

| Файл | Для чего |
| --- | --- |
| [`00-quickstart.md`](00-quickstart.md) | End-to-end старт: local plugin, custom tool, permissions, запуск, verification и npm packaging skeleton. |
| [`01-plugin-runtime.md`](01-plugin-runtime.md) | Как OpenCode находит, ставит и запускает plugins; порядок загрузки, entrypoints, options, context, ошибки. |
| [`02-hooks-reference.md`](02-hooks-reference.md) | Swagger-like справочник всех server hooks из `@opencode-ai/plugin`, включая недокументированные `chat.*`, `experimental.*`, `provider`, `auth`, `tool.definition`. |
| [`03-events-reference.md`](03-events-reference.md) | Справочник event bus/SSE events: официальные, скрытые, V2 `session.next.*`, envelope форматы и примеры подписки. |
| [`04-custom-tools-and-guards.md`](04-custom-tools-and-guards.md) | Custom tools, guard patterns, permission prompts, safe mutation of tool args/output, рецепты для hooks-focused разработки. |
| [`05-hidden-capabilities.md`](05-hidden-capabilities.md) | Недокументированные возможности из исходников: TUI plugins, slots, workspace adapters, provider/auth hooks, config mutation, flags. |
| [`06-dialogs-and-permissions.md`](06-dialogs-and-permissions.md) | Как вызывать UX-диалоги: TUI dialogs, выбор варианта ответа, question tool, permission approvals, SDK replies. |
| [`07-plugin-best-practices.md`](07-plugin-best-practices.md) | UX/performance/reliability стандарт для production plugins: нативный UI, async bootstrap, fail-open/fail-closed, case studies. |
| [`08-programmatic-slash-commands.md`](08-programmatic-slash-commands.md) | Как делать slash-style TUI команды, которые выполняют код напрямую через `api.keymap.registerLayer`, без LLM-turn и prompt rewrite. |
| [`09-rules-and-memory.md`](09-rules-and-memory.md) | Как OpenCode загружает memory/rules: `AGENTS.md`, `CLAUDE.md`, `opencode.json` `instructions`, и как раскладывать правила между global/project/workspace файлами. |
| [`10-desktop-development.md`](10-desktop-development.md) | Как устроен OpenCode Desktop: Electron, shared web UI, что можно менять plugin-ами, а что требует core/app PR. |

## Быстрый Выбор Поверхности Расширения

| Нужно сделать | Лучший механизм |
| --- | --- |
| Собрать первый local plugin/tool | [`00-quickstart.md`](00-quickstart.md). |
| Спроектировать production plugin UX | [`07-plugin-best-practices.md`](07-plugin-best-practices.md). |
| Блокировать опасные tool calls | `tool.execute.before` или `permission` config. |
| Менять аргументы tool call перед выполнением | `tool.execute.before`. |
| Пост-обработка результата tool call | `tool.execute.after`. |
| Добавить новый инструмент для модели | `tool` hook или файлы `.opencode/tools/*.ts`. |
| Разложить project memory/rules | [`09-rules-and-memory.md`](09-rules-and-memory.md): global `AGENTS.md` только для личных правил, project/workspace `AGENTS.md` плюс `opencode.json` `instructions` для shared rules. |
| Менять список сообщений перед LLM | `experimental.chat.messages.transform`. |
| Инжектить system prompt/context | `experimental.chat.system.transform` или `experimental.session.compacting`. |
| Менять provider params/headers | `chat.params`, `chat.headers`. |
| Добавить auth flow провайдера | `auth` hook. |
| Динамически подгружать модели провайдера | `provider.models`. |
| Сделать TUI slash-команду без LLM-turn | [`08-programmatic-slash-commands.md`](08-programmatic-slash-commands.md): TUI `api.keymap.registerLayer({ commands })` + `run()`. |
| Встраиваться в TUI | TUI plugin default export `{ id, tui(api) {} }`. |
| Менять Desktop/Web UI | Core change в `packages/app`; Desktop app в `packages/desktop` только оборачивает shared UI. TUI slots в Desktop не рендерятся. См. [`10-desktop-development.md`](10-desktop-development.md). |
| Слушать все события | server plugin `event`, SDK `client.event.subscribe()`, HTTP `/event`. |
| Управлять TUI извне | HTTP/SDK `tui.*` endpoints или TUI events `tui.*`. |
| Показать выбор/confirm/prompt в UI | TUI plugin `api.ui.dialog.replace(...)` + `DialogSelect/Confirm/Prompt/Alert`. |
| Спросить пользователя во время agent run | Built-in `question` tool. |
| Попросить разрешение на операцию | Permission config или custom tool `ctx.ask(...)`. |

## Главная Ментальная Модель

OpenCode расширяется на трех уровнях:

1. Server plugins: влияют на LLM loop, tools, permissions, providers, auth, config и server event bus.
2. Custom tools: добавляют callable tools для модели, через plugin `tool` или `.opencode/tools`.
3. TUI plugins: меняют интерфейс, команды, routes, slots, dialogs, themes и могут слушать события.

Desktop/Web UI не является четвертым user-land plugin surface. Его можно менять как часть upstream/fork разработки через `packages/app` и `packages/desktop`, но локальный TUI plugin не добавит элементы в Electron/Web интерфейс.

Server plugin hook, кроме `event` и object hooks, обычно имеет форму:

```ts
hook(input, output) => Promise<void>
```

OpenCode передает mutable `output`. Меняйте его in-place. Если hook бросает ошибку, действие обычно прерывается.

## Практический Минимум Plugin

```ts
import type { Plugin } from "@opencode-ai/plugin"

export default {
  id: "uplift.example",
  server: async ({ client, directory, worktree, $ }) => {
    await client.app.log({
      body: { service: "uplift.example", level: "info", message: "loaded" },
    })

    return {
      "tool.execute.before": async (input, output) => {
        if (input.tool === "read" && String(output.args.filePath).includes(".env")) {
          throw new Error("Reading env files is blocked by uplift.example")
        }
      },
    }
  },
} satisfies { id: string; server: Plugin }
```

Для local path plugins новый формат с default export object и `id` надежнее legacy named export functions, потому что TUI loader и plugin metadata завязаны на plugin id.
