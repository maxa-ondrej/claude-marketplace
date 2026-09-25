---
name: worktree
description: Opens an isolated herdr git worktree on a branch and launches a Claude agent inside it with a given prompt. Removes the worktree automatically when that agent exits. Tool-agnostic — the branch and prompt are given as arguments; this skill knows nothing about tickets or trackers.
argument-hint: <branch> [--base <ref>] [--label <text>] -- <prompt for the launched agent>
---

# Worktree Skill

Open a branch in an isolated [herdr](https://herdr.dev) worktree, launch a Claude agent in it with
a prompt, and tear the worktree down when that agent exits.

This skill **does not select work**. It takes a branch name and a prompt from `$ARGUMENTS`.
Whoever calls it (a tracker plugin, another skill, or the user) owns ticket selection, branch
naming, and status updates.

## Prerequisites

Requires `HERDR_ENV=1`:

```bash
test "${HERDR_ENV:-}" = 1 || { echo "not inside herdr"; exit 1; }
```

If it fails, tell the user this skill only runs inside a herdr session and stop.

## Arguments

`$ARGUMENTS` is `<branch> [--base <ref>] [--label <text>] -- <prompt>`.

- **branch** (required) — first token. The branch to check out; created from `--base` if it does
  not exist.
- **--base** *(optional)* — base ref for a new branch. Default `origin/main`.
- **--label** *(optional)* — herdr workspace label. Default: the branch name.
- **prompt** (required) — everything after `--`. The initial prompt the launched agent boots into.

If the branch is missing, ask the user for one. If the prompt is missing, ask what the agent
should do — do not invent one.

## Optional host-repo config

Read a `## Worktree` section from the host repo's CLAUDE.md (or AGENTS.md) if present. Both keys
are optional; with no section, skip straight to the launch.

| Key | Meaning |
|---|---|
| `seed` | Gitignored files a fresh checkout lacks, copied in from the main checkout — an inline list of paths, or the name of a file at repo root holding one path per line (e.g. `.worktreeinclude`) |
| `setup` | A shell command run inside the worktree before the agent starts (e.g. `direnv allow && pnpm install`) |

## Execution

### 1. Create the worktree

```bash
REPO=$(git rev-parse --show-toplevel)
herdr worktree create --cwd "$REPO" --branch <branch> --base <base> --label "<label>" --no-focus
```

From the JSON response capture:

- `.result.workspace.workspace_id` → **WS**
- `.result.root_pane.pane_id` → **PANE**
- `.result.worktree.path` → **WT**

### 2. Seed gitignored config

For each path from the `seed` config (skip silently if a path does not exist in `$REPO`):

```bash
cp -a "$REPO/<path>" "$WT/<path>"
```

Create parent directories as needed. Skip this step entirely if no `seed` config exists.

### 3. Launch the agent, with teardown chained after it

Write a launcher script — do **not** try to inline this, the nested quoting will bite you:

```bash
cat > "$WT/.herdr-launch.sh" <<EOF
cd "$WT"
<setup command, if configured>
rm -f "$WT/.herdr-launch.sh"
claude --append-system-prompt "\$(cat ${CLAUDE_PLUGIN_ROOT}/skills/worktree/agent-system-prompt.md)" "<prompt>"
cd "$REPO" && herdr worktree remove --workspace $WS --force
EOF
herdr pane run "$PANE" "bash $WT/.herdr-launch.sh"
```

Notes:

- The `--append-system-prompt` makes the launched agent speak briefly and ask questions
  interactively. Always pass it.
- The teardown line runs when the launched `claude` process exits, for any reason. It removes the
  checkout and closes the workspace. **The branch survives** — only the working copy goes.
- `--force` discards anything uncommitted in the worktree. The launched agent is responsible for
  committing and pushing before it exits.
- Do not `--focus`; keep the user where they are.

### 4. Report

Two or three lines, no more:

- Branch and worktree path
- Workspace ID (so the user can `herdr workspace focus <WS>` or remove it early)
- That the worktree removes itself when the agent exits, and uncommitted work in it is discarded

Then return. Do **not** block, poll, or wait on the launched agent — it runs autonomously.

## Removing a worktree early

If the user asks to close one before its agent finishes:

```bash
herdr worktree list --cwd "$(git rev-parse --show-toplevel)"
herdr worktree remove --workspace <WS> --force
```

Check `git -C <path> status --porcelain` first and warn if it is dirty.
