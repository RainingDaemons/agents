# Agents

This repo holds the AI agents structure I personally use across my projects. It bundles skills, sub-agents and commands so any new project can start with a consistent, well-tested toolkit instead of being rebuilt from scratch.

## What's inside

### `.agents/skills`

Audited skills, curated and reviewed before being added. Each folder contains a `SKILL.md` with the workflow and references for a specific domain - accessibility, API design, Rust/Python/Postgres patterns, security review, Svelte 5, and more. Skills are loaded on demand by the model when a task matches their description.

### `.opencode`

A multi-agent setup plus ready-to-use commands for [Opencode](https://opencode.ai).

- **Agents** (`agent/`): `chat` (read-only Q&A), `commit` (branch, commit, push), `review` (pre-commit standards + spec review), `test` (writes/runs tests, never edits production code).
- **Commands** (`commands/`): `document` (changelog + specification docs) and `ship` (orchestrates a feature end to end through gated stages).

Both agents and commands are wired to per-role models in `models.json`.

## Using with opencode

### Install the plugins

When working with agents, it is recommended to install the plugins declared in `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": [
    "opencode-models-manager@0.1.1",
    "opencode-agents-watcher@0.1.0"
  ]
}
```

Install them with opencode (run from the repo root) so the agents and commands resolve their models and get supervised correctly:

```bash
opencode
```

### Plugin-related required files

The following files inside `.opencode` are required by the plugins and must be present:

| File           | Purpose                                                        |
| -------------- | -------------------------------------------------------------- |
| `models.json`  | Models manager settings: maps each agent and command to a specific model.               |
| `monitor.json` | Agents watcher settings: timeout, polling, auto-restart policy. |
| `tui.json`     | Registers the agents-watcher plugin for the TUI.               |

- `models.json` - per-role model mapping used by `opencode-models-manager`:
- `monitor.json` - supervision policy read by `opencode-agents-watcher`.
- `tui.json` - loads the watcher plugin into the TUI.

## Quick start

1. Copy `.agents/skills`, `.opencode` and `opencode.json` into your project.
2. Install the plugins listed in `opencode.json`.
3. Keep `models.json`, `monitor.json` and `tui.json` in `.opencode`.
4. Start opencode and pick an agent or run a command.
