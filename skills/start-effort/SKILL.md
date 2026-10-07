---
name: start-effort
description: Start a multi-session effort in this session as its coordinator. On first use it also adds a short marked block to the user's ~/.claude/CLAUDE.md so every session follows the multi-session-effort rules.
disable-model-invocation: true
argument-hint: <one-line description of the effort>
---

You are starting a multi-session effort, and this session is its coordinator. The effort, as the user described it: $ARGUMENTS

## Step 1: the CLAUDE.md block (one-time setup)

Check whether the user-level `~/.claude/CLAUDE.md` contains the line `<!-- multi-session-effort:begin -->`. Use an exact text search (e.g. `grep -F`), not a judgment about whether similar rules are present.

- **Not found** (or the file doesn't exist): append the block below to the end of `~/.claude/CLAUDE.md`, creating the file if needed, with one blank line before it. Change nothing else in the file. Then tell the user, in one or two sentences, that you added it, and that from now on any session knows these rules, so they can start an effort just by saying so; `/multi-session-effort:start-effort` keeps working.
- **Found**: don't touch the file. Tell the user in one sentence that the block is already installed, so they don't need this command any more: saying "start an effort for …" in a session named `coordinator-g1` does the same. Then carry on, since they ran it.
- **Found `begin` without a matching `<!-- multi-session-effort:end -->`, or more than one `begin`**: don't edit. Show the user the lines you found and ask how to fix them.

The block:

```markdown
<!-- multi-session-effort:begin -->
## Multi-session efforts

When an effort runs across several Claude Code sessions that message each other,
load the `multi-session-effort` skill (multi-session-effort plugin) before naming,
launching or handing off a session. In short: sessions are named by role with a
generation tag (`coordinator-g1`, `implementor-1-g1`, `reviewer-1-g1`,
`out-of-band-<topic>-g1`); every role starts at `g1` in a new effort and the tag
goes up only at a handoff within the effort. Use the plugin's subagents by default
(`multi-session-effort:records-analyst`, `multi-session-effort:adversarial-verifier`,
`multi-session-effort:red-green-checker`, `multi-session-effort:number-tracer`,
`multi-session-effort:render-checker`).
Local playbooks and rules below this block take precedence where more specific.
<!-- multi-session-effort:end -->
```

## Step 2: load the rules and any local playbook

1. Load the `multi-session-effort` skill and follow it from here on.
2. Look in `~/.claude/CLAUDE.md`, the project's `CLAUDE.md` and the memory index for a local playbook or rules for this kind of effort (for example a file under `~/.claude/playbooks/`). If one applies, read it, and any backlog it points to, before going further. Where it's more specific than the skill, it wins.

## Step 3: confirm the effort and this session's name

1. If `$ARGUMENTS` is empty or too vague to name the effort, ask the user for a one-paragraph intake: who asked, what decision it feeds, the deadline, who sees the output, and the tracker issue if there is one.
2. This session should be named `coordinator-g1` (or `coordinator-gN` if it took over from an earlier coordinator in the same effort). If you can't tell its name, ask the user. If it isn't named yet, suggest `/rename coordinator-g1` now, before any peer exists.
3. Pick a short effort slug (`<effort-slug>`) (the tracker key, lowercased, or two or three words) and confirm it with the user along with the intake.

## Step 4: coordinator setup

Do section 2, step 3 of the skill:

- Write the role file `coordinator-gN-<effort-slug>-role.md` (for example `coordinator-g1-<effort-slug>-role.md`) in this session's memory dir: role, effort, the repos in scope (ask the user if it's not clear), roster (just this session for now; each entry gets name, role, owner, repos and worktrees), the user's decisions with today's date, and open items. Add a pointer in the memory index if there is one.
- Ask the user which shared resources the effort needs and who grants them, and record the comms rule.
- If anything will be measured, agree the headline definition before anything runs.

## Step 5: propose the first peers

Propose one implementor and one reviewer unless the work clearly needs something else. For each, give the user:

- the launch command, run from the right directory (`claude -n implementor-1-g1`);
- its worktrees, one per repo it will change (never this session's checkout or another peer's);
- its first message, filled in from the skill's template.

Don't launch sessions yourself or send peer messages until the user confirms.
