# Programmatic TUI Commands

Use this pattern when a slash-style TUI action should run local code immediately instead of turning into an LLM prompt. The practical example is `/worktree`: selecting it from the TUI slash autocomplete or command palette can open a TUI dialog, run local git/worktree code, and report status without creating an assistant turn.

Source note: current upstream keeps legacy `api.command.*` only as a v1 shim. New TUI plugins should use `api.keymap.registerLayer({ commands, bindings })`; `api.command.register`, `api.command.trigger`, and `api.command.show` are deprecated in `packages/plugin/src/tui.ts` and bridged through `packages/opencode/src/cli/cmd/tui/plugin/command-shim.ts`.

## Mental Model

OpenCode has several command surfaces that can look similar from the user seat:

| Surface | Runs Where | LLM Turn? | Use For |
| --- | --- | --- | --- |
| TUI `api.keymap.registerLayer({ commands })` | TUI plugin process | No | Local actions, navigation, dialogs, toasts, opening TUI routes, triggering scripts. |
| TUI `api.command.*` | TUI plugin process | No | Legacy v1 plugins only; avoid for new work. |
| Server slash/custom command plus `command.execute.before` | Server prompt flow | Yes | Mutating prompt parts before they are sent to the model. |
| HTTP/SDK `/tui/execute-command` and `/tui/publish` | External caller to active TUI | No | Remote-controlling an already running TUI. `/tui/execute-command` maps legacy aliases; use `/tui/publish` with `tui.command.execute` for arbitrary keymap command names. |

If the desired behavior is "user selects `/name`, local code runs, no assistant response is required", register a TUI keymap command with `namespace: "palette"` and `slashName`. Do not model this as a prompt template plus `command.execute.before`; that still enters the agent loop.

Important limitation: keymap command `run()` receives no raw slash text. If the action needs flags or free-form arguments, open `api.ui.DialogPrompt` or another TUI dialog from `run()`. A typed prompt like `/worktree -n 2` is not a stable argument transport for TUI keymap commands.

## Minimal Shape

```ts
import type { TuiPlugin } from "@opencode-ai/plugin/tui"

export default {
  id: "uplift.worktree-command",
  tui: async (api) => {
    api.keymap.registerLayer({
      commands: [
        {
          name: "uplift.worktree.create",
          title: "Create worktree",
          desc: "Create git worktree tab(s). Supports flags in the dialog.",
          category: "Worktree",
          namespace: "palette",
          slashName: "worktree",
          slashAliases: ["wt"],
          run() {
            api.ui.dialog.replace(() =>
              api.ui.DialogPrompt({
                title: "Create worktree",
                placeholder: "-n 2 --print --no-dirty",
                onConfirm: async (args) => {
                  api.ui.dialog.clear()
                  api.ui.toast({ variant: "info", message: "Creating worktree..." })

                  const result = await runWorktreeCommand({
                    directory: api.state.path.directory,
                    args,
                  })

                  api.ui.toast({
                    variant: result.status === 0 ? "success" : "warning",
                    message: result.message,
                    duration: 8000,
                  })
                },
                onCancel: () => api.ui.dialog.clear(),
              }),
            )
          },
        },
      ],
      bindings: [
        {
          key: "ctrl+w",
          cmd: "uplift.worktree.create",
          desc: "Create worktree",
        },
      ],
    })
  },
} satisfies { id: string; tui: TuiPlugin }
```

The exact command implementation can shell out, call a local TypeScript module, use the SDK, or open TUI UI. Keep the command item itself thin: gather user input, run one clear action, then surface the result.

`api.keymap.registerLayer(...)` is automatically scoped by the TUI plugin runtime: the returned cleanup is tracked and disposed when the plugin is deactivated or reloaded.

## Design Checklist

| Check | Why |
| --- | --- |
| Register from a TUI plugin with a stable `id`. | Local path TUI plugins rely on plugin identity for loading and enablement. |
| Use `api.keymap.registerLayer({ commands, bindings })`. | This is the current API; `api.command.*` is deprecated legacy shim code. |
| Include `name`, `title`, `category`, `namespace: "palette"`, and `slashName`. | Makes the command discoverable from the palette and slash autocomplete. |
| Implement `run()`. | This is the programmatic execution path; without it the command is only metadata. |
| Use `api.ui.dialog` for arguments. | TUI keymap `run()` does not receive raw slash arguments. |
| Use `api.state.path.directory` or `api.state.path.worktree` instead of `process.cwd()`. | The TUI process cwd is not always the target workspace. |
| Report progress and result with `api.ui.toast` or a dialog. | The command does not create an assistant turn, so user feedback must be explicit. |
| Keep expensive work out of registration. | Register metadata at plugin startup, but run filesystem/git/process work only inside `run()`. |
| Feature-detect only for compatibility shims. | New current API should assume `api.keymap`; only older-plugin compatibility should branch to `api.command`. |

## When Not To Use It

Use a normal slash/custom command or `command.execute.before` when the command's job is to prepare instructions for the model, review context, or produce an assistant answer.

Use a custom tool when the model should decide when to call the capability during an agent run.

Use a permission rule or `tool.execute.before` when the goal is to guard an operation rather than expose a user-invoked action.

Do not use TUI commands for Desktop/Web UI extensions. Desktop runs the shared app renderer from `packages/app`; TUI slots, dialogs, and keymap layers do not render inside the Electron/Web UI.

## Upgrade Smoke Test

After upgrading OpenCode or changing the plugin:

1. Start OpenCode with the TUI plugin enabled.
2. Open the command palette and confirm the command appears with the expected title/category.
3. Type `/worktree` and confirm slash autocomplete shows the TUI command.
4. Select the command and confirm `run()` executes without creating an LLM assistant turn.
5. If arguments are required, confirm the TUI dialog collects them and failure output is visible through a toast/dialog.
6. Confirm deactivating/reloading the plugin removes the command and keybinding.
