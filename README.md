# multi-session-effort

A Claude Code plugin for running one effort across several Claude Code sessions that message each other: a coordinator, implementors, reviewers and out-of-band side work.

It gives you:

- **A naming scheme that stays true.** Sessions are named by role with a generation tag (`coordinator-g1`, `implementor-1-g1`, `reviewer-1-g1`, `out-of-band-<topic>-g1`). Tags reset to `g1` for each new effort and go up only at a handoff within the effort.
- **A start-up and handoff routine.** The coordinator's setup checklist, a first-message template for each peer, the handoff protocol, and how to close an effort.
- **Subagents by default.** Five agent types (named `multi-session-effort:<agent>` once installed) that keep bulky reading and verification out of each session's context. None of them edits your project: `red-green-checker` works in its own throwaway worktree, and the others write only scratch files under `/tmp`. They are: `records-analyst`, `adversarial-verifier`, `red-green-checker`, `number-tracer`, `render-checker`.

## Install

In Claude Code:

```
/plugin marketplace add alphonso77/multi-session-effort
/plugin install multi-session-effort@multi-session-effort
```

Or from a shell (takes effect in new sessions, or after `/reload-plugins`):

```
claude plugin marketplace add alphonso77/multi-session-effort
claude plugin install multi-session-effort@multi-session-effort --scope user
```

## Use

Start the coordinator from the directory the work lives in, and run the start command:

```
claude -n coordinator-g1
> /multi-session-effort:start-effort <one-line description of the effort>
```

The first time, the command adds a short marked block to your user `~/.claude/CLAUDE.md` so every session knows to follow these rules. After that you can start an effort just by saying so; the command still works.

The coordinator then writes its role file, asks you for the shared resources and comms rules, and gives you the exact `claude -n …` commands for each peer, with each peer's first message.

## What's in the box

| Path | What it is |
|---|---|
| `skills/multi-session-effort/SKILL.md` | The rules: roles and names, starting an effort, handoffs, subagents by default, running rules, closing. |
| `skills/start-effort/SKILL.md` | The start command. Checks the `CLAUDE.md` block, then walks the coordinator through setup. |
| `agents/*.md` | The five read-only subagents. |

## Adding your own rules

Keep anything specific to your team (people, trackers, servers, lessons from past efforts) in a local playbook, e.g. `~/.claude/playbooks/<team>-effort-playbook.md`, and point to it from your `CLAUDE.md`. The skill reads a local playbook first, and where it's more specific, it wins.

## Requirements

- Claude Code with session names (`claude -n`) and messaging between sessions.
- `render-checker` needs Playwright installed in the project it checks.
- `red-green-checker` assumes a git repo and the project's test runner.

## License

MIT
