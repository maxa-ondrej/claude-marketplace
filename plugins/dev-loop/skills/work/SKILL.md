---
name: work
description: Run the whole development loop on one piece of work — research, plan, implement, review, ship — driving each step in order and stopping at the gates that need you. Use when the user hands over a story, bug, or feature and wants it taken end to end rather than one step at a time.
argument-hint: <story/bug description and task list>
---

# Work Skill

Take one piece of work from a description to a reviewed PR.

This skill **orchestrates**; it does no work of its own. Every step is another dev-loop skill, run
from this thread so the user can see each one land. It does **not** select the work — `$ARGUMENTS`
is the description, and whoever called it (a tracker plugin, or the user) owns the picking.

The handoffs are files in `.dev-loop/`, so any step can be re-run on its own later without
replaying the ones before it.

## Execution

### 1. Research — only if something is unfamiliar

Does this work depend on an API, library, or pattern that is not already obvious in the codebase?

- **Yes** → `/dev-loop:research <the open questions>`
- **No** → skip. Research that confirms what everyone already knows is waste.

### 2. Plan

`/dev-loop:plan $ARGUMENTS`

**This is a gate.** It ends with the user accepting, changing, or declining the plan. Do not walk
past it — an unapproved plan is not a plan, and everything downstream inherits its mistakes.

If the user declines, stop here and report why.

### 3. Implement

`/dev-loop:implement`

Builds the approved plan: tests first, then one developer per package.

### 4. Review and fix

`/dev-loop:review`

Then, while any `Blocker` or `Must Fix` finding is unchecked:

1. `/dev-loop:implement --fix`
2. `/dev-loop:review` again

Loop until clean. **No round limit** — but if the same finding survives three rounds, stop looping
and bring it to the user. Three failed attempts means the finding is wrong, the fix is wrong, or
the plan was wrong, and another round will not discover which.

### 5. Ship

`/dev-loop:ship`

Commits, pushes, opens the PR, watches CI, and pulls human review comments back in as findings —
which come back through step 4.

### 6. Report

The PR URL, what was built, and anything left open: findings deliberately skipped, review comments
answered rather than acted on, risks the plan flagged and accepted.

Keep it to a handful of lines. The detail is in `.dev-loop/` and in the PR.

## Stopping early

Stop and hand back to the user, rather than pushing on, when:

- The plan is declined
- A finding survives three fix rounds
- CI fails for a reason that is not in the diff (flaky infra, an unrelated break on main)
- A review finding contradicts AGENTS.md — a human decides which wins, not you

In every case say which step you stopped at and what is on disk, so the work can be picked up from
there.
