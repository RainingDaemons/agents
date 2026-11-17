# Agents

This repo holds the AI agents structure I personally use across my projects. It bundles skills, sub-agents and commands so any new project can start with a consistent, well-tested toolkit instead of being rebuilt from scratch.

## What's inside

### `.agents/skills`

Audited skills, curated and reviewed before being added. Each folder contains a `SKILL.md` with the workflow and references for a specific domain - accessibility, API design, Rust/Python/Postgres patterns, security review, Svelte 5, and more. Skills are loaded on demand by the model when a task matches their description.

### `.opencode`

A multi-agent setup plus ready-to-use commands for [Opencode](https://opencode.ai).

- **Agents** (`agent/`): `build` (implements features and fixes, never commits), `chat` (read-only Q&A), `commit` (branch, commit, push), `orchestrator` (drives `/ship`, delegates every stage to a subagent), `review` (pre-commit standards + spec review), `test` (writes/runs tests, never edits production code).
- **Commands** (`commands/`): `document` (changelog + specification docs) and `ship` (orchestrates a feature end to end through gated stages).

Both agents and commands are wired to per-role models in `models.json`.

## Using with opencode

### Install the plugins

OpenCode v2 separates server plugins from TUI-only plugins.

Server plugins go in the project `opencode.json` under `plugins`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugins": ["opencode-models-manager@0.2.1"]
}
```

The agents watcher is a **TUI-only** package: its `package.json` exports only `./tui`, with no server entrypoint. Declaring it in `opencode.json` makes the server fail with `Plugin entrypoint not found`. Add it to the global CLI config instead, at `~/.config/opencode/cli.json`:

```json
{
  "$schema": "https://opencode.ai/v2/cli.json",
  "plugins": ["opencode-agents-watcher@0.2.0"]
}
```

There is no project-local CLI config in v2, so this entry is global for your user. The TUI reloads `cli.json` on save; the watcher appears in the session sidebar. `opencode plugin add` targets the server config, so add a TUI-only package to `cli.json` by hand.

### Plugin-related required files

The following files inside `.opencode` are required by the plugins and must be present:

| File           | Purpose                                                        |
| -------------- | -------------------------------------------------------------- |
| `models.json`  | Models manager settings: maps each agent and command to a specific model.               |
| `monitor.json` | Agents watcher settings: timeout, polling, auto-restart policy. |

- `models.json` - per-role model mapping used by `opencode-models-manager`:
- `monitor.json` - supervision policy read by `opencode-agents-watcher`.

## Quick start

1. Copy `.agents/skills`, `.opencode` and `opencode.json` into your project.
2. Install the plugins listed in `opencode.json` (server plugins).
3. Add `opencode-agents-watcher@0.2.0` to your global `~/.config/opencode/cli.json` (TUI-only plugin).
4. Keep `models.json` and `monitor.json` in `.opencode`.
5. Start opencode and pick an agent or run a command.
