---
name: work
description: Fetches the next bug or story from the active Notion sprint (bugs always before stories, preferring in-progress work) and implements it. In the main checkout it hands the ticket off to an isolated worktree agent; inside a worktree it implements and ships the ticket end-to-end.
---

# Work Skill

Pick up a story or bug from the active sprint and get it done.

This skill has **two modes**, chosen by where it runs:

- **Main checkout** → select the ticket, then hand it to a fresh agent in an isolated worktree.
- **Linked worktree** → the branch is already checked out; implement and ship here.

## Step 0: Read the host repo's config

Read the `## Sprint` section of the host repo's CLAUDE.md (or AGENTS.md). If it is missing, STOP
and tell the user what to add. See the `agile-coach` agent for the expected keys — this skill
just hands `$ARGUMENTS` to it and otherwise delegates.

## Step 1: Detect the mode

```bash
test -f .git && echo worktree || echo main
```

`.git` as a **file** means a linked worktree → go to **Mode B**. A directory → **Mode A**.

---

## Mode A — main checkout: hand off to a worktree

### A1. Select the ticket

Invoke the `agile-coach` agent to:

- Find the active sprint
- Select the next **not-started** bug or story (pass `$ARGUMENTS` if the user named one)
- Update statuses to In Progress
- Report the **branch name** it would use and the ticket's **page ID** (32-hex UUID)

**Skip in-progress tickets.** This mode spins up a *new* worktree and agent; an already-in-progress
ticket usually has both already. Instruct the agile-coach to consider only not-started tickets —
unless the user named one via `$ARGUMENTS`, in which case honor it regardless of status.

The agile-coach must **not** run any `git checkout`/`git branch` commands — the worktree creates
the branch itself.

If no actionable ticket remains, report that and stop.

### A2. Open the worktree

Invoke the worktree plugin's `/worktree` skill:

```
/worktree <branch> --label "<ticket title>" -- /notion-sprint:work <ticket-page-id>
```

The launched agent re-selects that exact ticket inside the worktree and runs **Mode B** on it.

### A3. Report

Ticket title, branch, worktree path, and that the agent is running autonomously. Two or three
lines. Then stop — do not follow the worktree agent's progress.

---

## Mode B — inside the worktree: implement and ship

### B1. Pick up the ticket

Invoke the `agile-coach` agent with `$ARGUMENTS` (the ticket page ID) to fetch the ticket, its
tasks, and its description, and to move all statuses to In Progress. It will detect the worktree
and leave git alone — the branch is already checked out.

### B2. Do the work

Invoke the dev-loop plugin's `/dev-loop:work` skill with the ticket's description and task list.

That runs the whole loop — research if needed, plan (which stops for your approval), implement,
review and fix until clean, then ship. It reports where it stopped if it stops early.

### B3. Done

Invoke the `agile-coach` agent to update final statuses, then report:

- PR URL
- Tasks completed and their statuses
- Any review comments addressed or intentionally skipped
- Any remaining blockers

Everything must be **committed and pushed** before this skill returns — the worktree is removed
when this agent exits, and uncommitted work in it is discarded.
