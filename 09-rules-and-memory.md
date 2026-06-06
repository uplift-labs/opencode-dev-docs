# Rules And Memory

Практическая модель того, как OpenCode загружает долгоживущую память и инструкции.

Проверено по официальным страницам `Rules` и `Config` на `https://opencode.ai/docs` на 2026-05-08.

## Ментальная Модель

OpenCode memory - это file-based rules, а не скрытая база знаний, которая автоматически пополняется из чатов.

| Layer | Где хранится | Для чего |
| --- | --- | --- |
| Global rules | `~/.config/opencode/AGENTS.md` | Личные, долговременные предпочтения пользователя, применимые почти везде. |
| Project rules | `AGENTS.md` в project/repo/workspace дереве | Правила конкретного проекта, команды, архитектуры, workflow. |
| Claude fallback | `CLAUDE.md`, `~/.claude/CLAUDE.md` | Совместимость с Claude Code, когда соответствующего `AGENTS.md` нет. |
| Config instructions | `opencode.json` поле `instructions` | Явное подключение дополнительных markdown-файлов, glob patterns или remote URLs. |

Session history и summaries помогают внутри текущей/продолженной сессии, но не заменяют `AGENTS.md`. Если правило должно пережить новые сессии, его нужно записать в global/project rules или подключить через `instructions`.

## Как OpenCode Ищет `AGENTS.md`

При старте OpenCode ищет local rules, поднимаясь вверх от текущей директории:

1. Ищет `AGENTS.md` или fallback `CLAUDE.md`.
2. Первый найденный local rules-файл выигрывает.
3. Остальные ancestor `AGENTS.md` выше по дереву не объединяются автоматически.
4. Global `~/.config/opencode/AGENTS.md` добавляется отдельно и действует во всех сессиях.
5. Global `~/.claude/CLAUDE.md` используется как fallback, если нет `~/.config/opencode/AGENTS.md`.

Практическое следствие: если `workspace/AGENTS.md` содержит общие правила, а `workspace/repo/AGENTS.md` содержит repo-specific правила, запуск из `workspace/repo` подхватит repo-local файл и не подхватит parent файл сам по себе.

## Надежное Подключение Parent Rules

Используйте project `opencode.json` с `instructions`, когда child repo должен всегда получать parent/workspace правила:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "instructions": ["../AGENTS.md"]
}
```

`instructions` объединяются с найденными `AGENTS.md` rules. Пути и glob patterns в config-relative fields задавайте относительно файла `opencode.json`, который их объявляет.

Для monorepo/workspace можно подключать несколько файлов:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "instructions": [
    "../AGENTS.md",
    "docs/development-guidelines.md",
    "packages/*/AGENTS.md"
  ]
}
```

Используйте `instructions` для правил, которые должны грузиться всегда. Если файл нужен редко, лучше оставить явную инструкцию в `AGENTS.md` читать его по необходимости, чтобы не раздувать контекст каждой сессии.

## Где Держать Правила

| Rule type | Лучшее место |
| --- | --- |
| Личные устойчивые привычки пользователя | Global `~/.config/opencode/AGENTS.md`. |
| Организационные/workspace правила для группы repo | Parent/workspace `AGENTS.md`, подключенный child repos через `opencode.json` `instructions`. |
| Build/test commands, repo architecture, local gotchas | Repo root `AGENTS.md`, committed to git. |
| Package/module-specific exceptions | Ближайший repo/package `AGENTS.md`, если запуск часто идет из этого дерева; иначе отдельный doc через `instructions` или lazy reference. |
| Runtime/plugin configuration | `opencode.json`, not `AGENTS.md`. |
| Secrets, tokens, one-off debugging notes | Нигде в rules; используйте env vars, secret files, issue comments или task notes. |

Global file должен оставаться коротким. Если правило упоминает конкретный customer, remote, CLI, toolchain, repo path, build system или domain vocabulary, это почти всегда project/workspace rule, а не global memory.

## Управление Несколькими `AGENTS.md`

- Не рассчитывайте на автоматическое объединение parent и child `AGENTS.md`; добавляйте `opencode.json` `instructions`, если нужны оба.
- Не дублируйте один и тот же текст в parent и child files; вынесите shared часть вверх и подключите ее явно.
- Держите `AGENTS.md` компактным: команды, инварианты, проверочные шаги, known gotchas. Длинные справочники лучше подключать отдельными docs через `instructions` или читать lazy.
- Проверяйте перенос правил поиском по workspace, чтобы stale копии не остались в global/project files.
- После изменения `opencode.json` проверьте `opencode debug config` из нужной директории: в resolved config должно быть видно поле `instructions`.
- После изменения `AGENTS.md` начинайте новую сессию или явно перечитывайте файл; уже запущенная модель могла получить старый rules context.

## Пример: Workspace + Repo

```text
D:\work\mekha\
  AGENTS.md                 # GitFlic/workspace правила
  mcsetup\
    AGENTS.md               # MCSetup build/test/Qt правила
    opencode.json           # Подключает ../AGENTS.md
```

`D:\work\mekha\mcsetup\opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "instructions": ["../AGENTS.md"]
}
```

Такой layout избегает загрязнения global memory и гарантирует, что запуск OpenCode из `mcsetup` получит и workspace GitFlic правила, и repo-specific MCSetup правила.
