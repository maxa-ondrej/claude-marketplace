---
name: ship
description: Commit, push, open a PR, watch CI, and pull review comments back in as findings. The delivery loop from local changes to a reviewed PR. Use whenever the user asks to commit, push, ship, send, or submit changes.
---

# Ship Skill

Get the work out of the working tree and into a reviewed PR.

**Never push directly to `main`.** Everything goes through a feature branch and a PR.

## Execution

### 1. Check there is something to ship

`git status` and `git diff`. Nothing changed? Say so and stop.

### 2. Update docs and agent config

If the project has a `docs` agent, or the authoring plugin's `meta` agent is available, run them —
**in parallel, one message** — to update documentation and AGENTS.md from the diff. Stage whatever
they change.

Skip silently if neither exists.

### 3. Run the checks

Run the repo's format, codegen, typecheck, and test commands (from AGENTS.md). All four must pass.
Stage anything format or codegen rewrote.

### 4. Review before pushing

Run `/dev-loop:review`. If it leaves unchecked `Blocker` or `Must Fix` findings, run
`/dev-loop:implement --fix` and come back to step 3.

Shipping a known must-fix and hoping the PR reviewer catches it is how it reaches main.

### 5. Commit

- Stage the relevant files. Never `.env`, credentials, or `.dev-loop/`
- One concise message saying **why**, not what. Use a HEREDOC to preserve formatting
- If the user gave a message in `$ARGUMENTS`, use it
- No `Co-Authored-By`, `Generated-By`, or AI attribution footers

### 6. Push and open the PR

```bash
git push -u origin "$(git branch --show-current)"
```

Then `gh pr create`, unless a PR already exists. If the project's tracker workflow defines a
PR-body convention (a Notion/Jira link, say), follow it — the lookup itself belongs to that tracker
plugin, not here.

```bash
gh pr create --title "<short title>" --body "$(cat <<'BODY'
## Summary
- <bullets>

## Test plan
- <how it was verified>
BODY
)"
```

Return the PR URL.

### 7. Watch CI and comments (background)

Launch **both** in the background (`run_in_background: true`) and do not poll or sleep:

1. `gh run watch`, then `gh run view --log-failed` if it fails
2. Poll the PR's reviews and comments for up to six minutes

You will be notified when each finishes.

### 8. Handle what comes back

- **CI failed** → read the logs, fix, restart from step 3
- **Comments arrived** → `/dev-loop:review --pr` to triage them into findings, then
  `/dev-loop:implement --fix`
- **Green, no comments** → done

### 9. Report

PR URL and CI status. Do **not** touch tracker statuses — that belongs to the tracker plugin.

## Versioning

Nothing version-related goes in the PR. Releases happen after merge: a release tag triggers the
image build, and MajNet consumes it (see the majksa-deploy plugin's `/deploy`). No changesets, no
version bumps.
