---
name: ship
description: Commit, push, open a PR, watch CI, and pull review comments back in as findings. The delivery loop from local changes to a reviewed PR. Use whenever the user asks to commit, push, ship, send, or submit changes.
---

> **Talk to the user in ASD-STE100 Simplified Technical English.** Approved words only. One idea
> per short sentence. Active voice. The same word for the same thing every time. No jargon, no
> idioms, no long noun groups. This governs what you say, never what the code or the commit says.

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

**Use the repo's own verification agents when it has them.** Look in `.claude/agents/` at the repo
root and in the package directory you are verifying. A repo that ships an agent for its linter,
type checker or test runner knows something the raw command does not — how to read its output, and
how to fix what it finds. Invoke those agents in parallel, one message, instead of the bare
commands. Run the bare commands only for what no agent covers.


### 4. Review before pushing

Run `/dev-loop:review`. If it leaves unchecked `Blocker` or `Must Fix` findings, run
`/dev-loop:implement --fix` and come back to step 3.

Shipping a known must-fix and hoping the PR reviewer catches it is how it reaches main.

### 5. Commit

- Stage the relevant files. Never `.env`, credentials, or `.dev-loop/`
- One concise message saying **why**, not what. Use a HEREDOC to preserve formatting
- **Plain English, as short as possible.** Short subject line; bullets for the body whenever
  there is more than one point, prose only when a bullet cannot carry it. Plain words over
  jargon. Never restate the diff or pad the message to look thorough
- If the user gave a message in `$ARGUMENTS`, use it
- No `Co-Authored-By`, `Generated-By`, or AI attribution footers

### 6. Push and open the PR

```bash
git push -u origin "$(git branch --show-current)"
```

Then open the PR per `FORGE.md` (**Open a PR**), unless one already exists — `gh` on GitHub,
`glab` on GitLab. If the project's tracker workflow defines a PR-body convention (a Notion/Jira link, say), follow it — the lookup itself belongs to that tracker
plugin, not here.

For the body, run `/dev-loop:describe`. It writes the description from the commits, the diff and
the plan, shows it to the user, and hands it back here to open the PR with. Do not write a body
inline — one skill owns that format.

Return the PR URL.

### 7. Watch CI and comments (background)

Launch **both** in the background (`run_in_background: true`) and do not poll or sleep:

1. Watch CI per `FORGE.md`, and read the failed logs if it fails
2. Poll the PR's reviews and comments for up to six minutes

You will be notified when each finishes.

### 8. Handle what comes back

- **CI failed** → read the logs, fix, restart from step 3
- **Comments arrived** → `/dev-loop:address-review`, which triages, fixes, ships, and answers the
  threads. Only reaching for the pieces? `/dev-loop:review --pr` then `/dev-loop:implement --fix`

When a review round's fixes are committed, run `/dev-loop:tidy-history --round`. It folds that
round's commits into one named commit, so the branch does not reach merge as a list of `fix test`
and `address review`. Round mode leaves every reviewed commit alone, so the open threads keep their
anchors.
- **Green, no comments** → done

New commits went up since the description was written — a fix round, a CI repair? Run
`/dev-loop:describe` again. A description that describes an older version of the branch misleads
every reviewer who reads it.

### 9. Report

PR URL and CI status. Do **not** touch tracker statuses — that belongs to the tracker plugin.

## Versioning

Nothing version-related goes in the PR. Releases happen after merge: a release tag triggers the
image build, and MajNet consumes it (see the majksa-deploy plugin's `/deploy`). No changesets, no
version bumps.
