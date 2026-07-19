---
name: worktree
description: Selects the next ticket (or a specified one) and opens it in an isolated herdr-managed git worktree — creates the branch, seeds gitignored config, runs direnv allow + install, and launches a Claude agent inside it. Use this to spin up parallel work on a ticket without disturbing the main checkout.
---

# Worktree Skill

Pick up a ticket and open it in an isolated [herdr](https://herdr.dev) worktree with the dev
environment ready and a fresh Claude agent running. Unlike a full work loop, this does **not**
implement the ticket — it prepares an isolated workspace so a dedicated agent can.

## Step 0: Read the host repo's config

Read the `## Worktree` section of the host repo's CLAUDE.md (or AGENTS.md). If it is missing,
STOP and tell the user what to add. Expected keys:

| Key | Meaning | Example |
|---|---|---|
| `script` | Path to this repo's herdr bootstrap script (creates/reuses the worktree, seeds config, installs deps, launches the agent) | `scripts/herdr-worktree.sh` |
| `seed_paths` | Gitignored files this repo needs seeded into a fresh worktree (hand-maintained config a fresh checkout lacks — not machine-generated dirs like `node_modules/`) — or a note that they're read from a `.worktreeinclude`-style file at repo root, one path per line | `.env.local`, `.env.preview.local`, `.claude/settings.local.json` |
| `install_cmd` *(optional)* | Dependency install command the script runs after `direnv allow`. Defaults to the repo's package manager install | `pnpm install` |

If the repo has no such script yet, STOP and tell the user this skill needs one — point them at
another repo's `scripts/herdr-worktree.sh` as a starting template (creates/reuses a worktree via
`herdr worktree create`, resolves it with `herdr worktree list --json`, copies `seed_paths` in,
runs `direnv allow` + the install command, then launches `herdr` with an initial prompt).

## Execution

Follow these phases **in order**. Stop and report if any phase fails. Pass `$ARGUMENTS` to the
ticket-selection step if the user specified a ticket.

### Phase 1: Select the ticket

Ticket selection is pluggable:

- **If the notion-sprint plugin is installed**, invoke its `agile-coach` agent via the Agent tool to:
  - Find the active sprint
  - Select the next **not-started** bug or story (pass `$ARGUMENTS` if the user named one)
  - Update statuses to In Progress
  - **Determine the branch name** it would use (e.g. `feat/...` or `fix/...`)
  - **Report the selected ticket's page ID** (for Notion, the 32-hex UUID)

  **Skip in-progress tickets.** Unlike a full work loop, this skill spins up a *new* dedicated
  agent in a *new* worktree — an already-in-progress ticket almost always already has a worktree
  and agent on it. So instruct the agile-coach to **only consider not-started tickets** (bugs
  before stories, stories by epic number, per its own config) and to **skip anything already In
  Progress**. If the user named a specific ticket via `$ARGUMENTS`, honor that regardless of
  status. If no not-started ticket remains, the agile-coach should report that and Phases 2/3 are
  skipped.

  **Important:** Instruct the agile-coach to **NOT run any `git checkout`/`git branch` commands**
  and **NOT touch the main checkout** — the worktree script creates the branch from `origin/main`
  itself. The agile-coach should only report the branch name (new or, for resumed work, the
  existing branch) and the ticket's identifier.

- **Otherwise (no ticket-tracking plugin installed)**, take the ticket/branch name directly from
  `$ARGUMENTS`. If not provided, ask the user for a branch name and a short label. There is no
  status to update and no page ID to pass through — the launched agent's initial prompt should
  describe the work directly instead of referencing a ticket ID.

Review the work summary and capture the **branch name** and (if applicable) the **ticket
identifier**.

### Phase 2: Open the worktree

Run the repo's bootstrap script (`<script>` from the config) with the branch name, the ticket
title as the label, and a `--prompt` that makes the launched Claude pick up the ticket
immediately:

```bash
<script> <branch> --label "<ticket title>" --prompt "<initial command for the launched agent>"
```

Compose the `--prompt`:
- If notion-sprint selected the ticket, use its work skill with the ticket's page ID (e.g.
  `/work <ticket-page-id>`) so the worktree agent re-selects that exact ticket and implements it
  end-to-end.
- Otherwise, describe the task directly (e.g. `/implement <description from $ARGUMENTS>`).

This creates (or reuses) a herdr worktree on the branch, seeds the gitignored config files from
`seed_paths`, runs `direnv allow` + the install command, and launches a focused Claude agent
inside the worktree that boots straight into the composed prompt. (If notion-sprint's agile-coach
is worktree-aware, it skips the `main` checkout/branch creation when run inside a linked worktree,
since the branch is already checked out there.)

Useful flags (check the script's own `--help`, conventionally): `--no-install` (skip deps),
`--no-agent` (prepare only), `--base <ref>` (base a new branch on something other than
`origin/main`), `--prompt <text>` (override the opening command).

### Phase 3: Report

Present:
- Ticket title and status (if ticket-tracked)
- Branch name
- Worktree path
- That a Claude agent was launched and focused in herdr, now running the composed prompt (or any
  step that was skipped/failed)

The new agent runs autonomously inside the worktree — implementing and shipping the ticket without
further input from this session.
