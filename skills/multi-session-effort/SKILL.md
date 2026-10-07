---
name: multi-session-effort
description: Rules for an effort run across several Claude Code sessions that message each other (a coordinator, implementors, reviewers, out-of-band side work). Covers role-based session names with per-effort generation tags (coordinator-g1, implementor-1-g1), handoffs, the coordinator's setup, the first message to each peer, using subagents by default, and closing an effort. Use it when starting, joining, handing off or closing a multi-session effort, when naming or launching a peer session, or when a session's role file says it is part of one.
---

# Multi-session efforts

An **effort** is one piece of work (a feature, an investigation, a release) run across several Claude Code sessions that message each other. These rules keep the sessions' names stable, their ownership clear, and their checking cheap.

**Local rules come first.** If the user's `CLAUDE.md` or memory names a local playbook for this kind of effort (people, trackers, shared resources, lessons from past efforts), read it before acting. Where it's more specific than this skill, it wins.

## 1. Roles and names

Name sessions by **role**, not by the task they were started for. A name like `login-bugfix-reviewer` goes stale as soon as that task ends; a role name stays true for the whole effort.

| Role | Name | Owns |
|---|---|---|
| Coordinator | `coordinator-gN` | Scheduling, shared resources (test boxes, feature flags, deploy workflows, contract versions), decisions, the roster, and talking to peers. One at a time. |
| Implementor | `implementor-K-gN` | Writing code and opening PRs, in its own worktree. Add more as the work splits. |
| Reviewer | `reviewer-K-gN` | Reviewing PRs, by session message. Add more as needed. |
| Out-of-band | `out-of-band-<topic>-gN` | Side work that isn't part of the main effort. Not polled or handed off with it. |

`K` numbers parallel sessions of the same role. `gN` is the **generation tag**:

- Every role starts at **`g1` when a new effort starts.**
- Bump `gN` only when that role **hands off to a fresh session within the effort** (`coordinator-g1` → `coordinator-g2`).
- Add the tag deliberately. Claude Code adds a random two-word suffix when two **live** sessions want the same name. That's a safety net, not the scheme.

How names behave in Claude Code:

- Name a session at launch: `claude -n <name>` (long form `--name`). That's better than `/rename` later, because peers never see a temporary name.
- `claude --resume <name>` opens that session. If several sessions share the name, it opens the picker filtered to it. Because tags reset per effort, `--resume coordinator-g1` may list every past effort's coordinator; pick by date. Messaging between peers is unaffected, because only live sessions are addressed.
- A peer is addressed by its current name. **Rename only at a handoff**, and send every peer the old → new mapping in one message.
- The name doesn't say what a session is working on, so each session keeps a role file (in its memory dir) that starts with its role, the effort, and its open items.

## 2. Starting an effort

`/multi-session-effort:start-effort <effort>` walks the coordinator through this.

1. **Intake.** Write the ask in one paragraph: who asked, what decision it feeds, the deadline, and who sees the output. Open or choose a tracker issue; requirements and acceptance criteria go in the issue **description**, not only in comments.
2. **Launch the coordinator** from the directory the work lives in (it picks the memory dir): the repo for repo work, or the home directory for cross-repo work. `claude -n coordinator-g1`.
3. **Coordinator setup:**
   - Write a role file named for the effort, `coordinator-gN-<effort>-role.md` (e.g. `coordinator-g1-<effort>-role.md`) (role names repeat across efforts, so the effort goes in the file name): role, effort, roster (name, role, owner), the user's decisions with dates, and open items. Add a pointer in the memory index if there is one.
   - List the shared resources the effort needs and who grants them. Only the coordinator schedules them.
   - Write the comms rule up front: nothing goes outside the team until it's reviewed, the user decides when, and no commitments or dates are made on the user's behalf.
   - For anything measured, agree the **headline definition** before running anything: which metric, which population, and whether it's a mean across runs or a single run.
4. **Launch peers only when the coordinator asks for them**, starting with one implementor and one reviewer; add more only when the work splits cleanly. The coordinator gives the user the exact launch commands (`claude -n implementor-1-g1`, `claude -n reviewer-1-g1`) and writes each peer's first message.

### First message to each peer (template)

> You are <name>, <role> for <effort> (<tracker key>). The coordinator is <coordinator name>. The user merges every PR and runs every deploy.
> Work in your own worktree, never in <shared checkout>. Shared resources (<list>) belong to the coordinator: ask before touching them.
> Reviews happen by session message, and each round is also posted on the PR as a review with the verdict on its first line. Check final CI before calling anything ready.
> **Use subagents by default** (multi-session-effort skill, section 4). In each status message, say which checks ran as subagents.
> Your first task: <task>. Keep a role file that starts with your role, the effort and your open items.

## 3. Handoffs

- Before a handoff, the coordinator asks the other peers whether they need one too, so handoffs happen together.
- The outgoing session writes a handoff file into a memory dir (copy it out of `/tmp` if it started there): role, effort, roster, decisions, open items, and anything in flight.
- The new session starts with the next generation tag (`implementor-1-g2`), reads the handoff file, and confirms **"handoff received"** before the old session steps back.
- The coordinator sends every peer the old → new name mapping in one message.

## 4. Subagents by default

Every session may spawn subagents (the Agent tool), and should, rather than doing bulky or independent checks inline. Doing them inline bloats the session's context and lets mistakes through: a grep that misses a multi-line sentence, an overclaim caught only in a second review round.

This plugin ships five agent types. Their full names, for the Agent tool, carry the plugin prefix: `multi-session-effort:records-analyst` and so on.

| Agent | Use it for |
|---|---|
| `records-analyst` | Reading more than ~3 files, or any raw records, logs, CSVs or git history. Recomputes figures from source. |
| `adversarial-verifier` | One per blocker, should-fix, or claim, before it's posted or sent. Tries to disprove it. |
| `red-green-checker` | Every review round of a fix PR: new tests fail on base for the right reason and pass on head. |
| `number-tracer` | Any text others will rely on (readouts, pages, emails, PR bodies that quote figures), before it ships. |
| `render-checker` | Any static HTML page, before it ships. Needs Playwright in the project. |

Patterns:

| Situation | Pattern |
|---|---|
| Many near-identical verifications (per run, per leg) | One `records-analyst` each, in parallel; merge the verdict tables. |
| A review round | In parallel: the reading review in the main context, an `adversarial-verifier` per finding, a `records-analyst` recompute of each headline number, and `red-green-checker` on a fix PR. |
| Before sending a claim to the coordinator or the user | An `adversarial-verifier` on the claim. |
| Text with numbers going to others | `number-tracer` on the final text. |
| Deploy checks, migration gates, recounts, file diffs | A `records-analyst` returning pass/fail plus numbers. |
| Coordinator status, backlog and inventory sweeps | Parallel subagents, one per source. |

Limits:

- Subagents never touch shared resources (test boxes, flags, merges, tracker or GitHub writes) or another session's checkout.
- They may be denied file writes, so have them return results as text.
- Their messages go out under the parent session's name; they aren't peers.
- The parent session stays accountable: it checks a subagent's key claims against the source before acting on or reporting them.

Peers remain the right tool for work that needs its own worktree, a review loop or a long-lived role. Subagents are for bounded, read-mostly tasks that return a verdict.

## 5. Running rules

- The user merges every PR and dispatches every deploy. Sessions never merge, deploy or log into servers unless the user says so.
- Never enter credentials or complete a sign-in on the user's behalf. Never read local secret files (`.env*.local` and the like).
- Freeze the code (one SHA, clean-tree checks) for any measurement set, and lift the freeze explicitly.
- Every number in outward-facing text traces to a merged readout or an independent recompute.
- Separate MEASURED from INFERRED, and say "likely" when the cause isn't isolated. State the scope (which config, which version, which population).
- Hold comms until reviewed. When a correction is needed, own it plainly.
- Check final CI (not pending) before saying "ready".
- Don't overwrite a file the user edited; take their version as current.
- Ask before deleting or moving anything shared.

## 6. Closing an effort

- The coordinator asks every peer to update its role file ("effort complete <date>" and open items), remove its worktrees, and report its open items and where subagents would have helped.
- The coordinator writes a backlog file for the next effort and folds the lessons into the local playbook, if there is one.
- Out-of-band sessions are told separately.
- The next effort starts fresh sessions at `g1` rather than resuming these.
