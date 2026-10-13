---
name: worktree
description: Opens an isolated herdr git worktree on a branch and launches a Claude agent inside it running /dev-loop:work. Removes the worktree automatically when that agent exits. The branch is given as an argument; an optional trailing prompt overrides the default /dev-loop:work. Knows nothing about tickets or trackers itself.
argument-hint: <branch> [--base <ref>] [--label <text>] [-- <prompt override>]
---

# Worktree Skill

Open a branch in an isolated [herdr](https://herdr.dev) worktree, launch a Claude agent in it
running `/dev-loop:work`, and tear the worktree down when that agent exits.

This skill **does not select work** itself. It takes a branch name from `$ARGUMENTS` and boots the
launched agent into `/dev-loop:work`, which picks the work up on the other side. Whoever calls it
(a tracker plugin, another skill, or the user) owns branch naming and status updates.

## Prerequisites

Requires `HERDR_ENV=1`:

```bash
test "${HERDR_ENV:-}" = 1 || { echo "not inside herdr"; exit 1; }
```

If it fails, tell the user this skill only runs inside a herdr session and stop.

## Arguments

`$ARGUMENTS` is `<branch> [--base <ref>] [--label <text>] [-- <prompt override>]`.

- **branch** (required) — first token. The branch to check out; created from `--base` if it does
  not exist.
- **--base** *(optional)* — base ref for a new branch. Default: the repo's **own** default
  branch, read from `refs/remotes/origin/HEAD` (see step 1). Never assume `origin/main` — it is
  `origin/master` in some repos and branching from the wrong base silently produces a huge
  bogus diff.
- **--label** *(optional)* — herdr workspace label. Default: the branch name.
- **prompt** *(optional)* — everything after `--`. Overrides what the launched agent boots into.
  Omit it and the agent starts `/dev-loop:work`.

If the branch is missing, ask the user for one. A missing prompt is **not** an error and needs no
question — that is the normal case, and the agent boots into `/dev-loop:work`. Only pass an
override when the caller explicitly supplied one.

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

# Resolve the base ref. Use the caller's --base if given, otherwise detect the repo's
# default branch. Do NOT hardcode a guess.
BASE="<--base value if the caller supplied one, else empty>"
if [ -z "$BASE" ]; then
  BASE=$(git -C "$REPO" symbolic-ref --short -q refs/remotes/origin/HEAD)
  if [ -z "$BASE" ]; then
    # origin/HEAD is unset in this clone; repair it from the remote, then retry.
    git -C "$REPO" remote set-head origin --auto >/dev/null 2>&1
    BASE=$(git -C "$REPO" symbolic-ref --short -q refs/remotes/origin/HEAD)
  fi
fi
if [ -z "$BASE" ]; then
  echo "cannot determine the default branch - rerun with an explicit --base <ref>" >&2
  exit 1
fi

herdr worktree create --cwd "$REPO" --branch <branch> --base "$BASE" --label "<label>" --no-focus
```

Fail loudly rather than falling back to a literal. A wrong base is not a small error: the new
branch forks from the wrong history and every later diff, MR and review is against the wrong
baseline.

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
claude --dangerously-skip-permissions --append-system-prompt "\$(cat ${CLAUDE_PLUGIN_ROOT}/skills/worktree/agent-system-prompt.md)" "<prompt>"
cd "$REPO" && herdr worktree remove --workspace $WS --force
EOF
herdr pane run "$PANE" "bash $WT/.herdr-launch.sh"
```

Notes:

- `<prompt>` is the override if the caller gave one after `--`, otherwise the literal
  `/dev-loop:work`. Substitute the chosen text into the launcher before writing it — do not emit
  the placeholder.
- `--dangerously-skip-permissions` is what lets the agent run unattended — it is launched to work
  on its own, and a permission prompt nobody is watching just stalls it. It is tolerable **here
  specifically** because the agent is confined to a disposable worktree on its own branch: the
  blast radius is a checkout that was going to be deleted anyway. Do not carry this flag into a
  main checkout.
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
