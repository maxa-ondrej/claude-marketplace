---
name: complete
description: Mark a story or bug as complete after its PR has been merged. Moves stories to their terminal "done" status and bugs to their terminal "fixed" status. Use when the user says a PR was merged or asks to complete/close a work item.
---

# Complete Skill

Mark work as complete after its PR has been merged.

## Step 0: Read the host repo's config

Read the `## Sprint` section of the host repo's CLAUDE.md (or AGENTS.md). If it is missing, STOP
and tell the user what to add. See the `agile-coach` agent for the expected keys — this skill
needs the status lifecycle mapping (which status is "story done" vs "bug fixed").

## Execution

1. Verify the PR is actually merged:

```bash
gh pr view --json state,body -q '{state: .state, body: .body}'
```

If the PR is not merged, tell the user and stop.

2. Extract the work-item page URL from the PR body (look for a `Notion: <url>` line, or whatever
   marker convention this repo's `/ship` skill uses, near the top). Use this URL to fetch and
   update the work item directly — no searching needed.

3. Fetch the page to determine its type (story or bug), then update its status per the `## Sprint`
   config's lifecycle mapping — typically:
   - Move the **story** to its terminal "done" status (e.g. `Done`)
   - Move the **bug** to its terminal "fixed" status (e.g. `✅ Fixed`)

4. Return to `main` — but only in the main checkout. In a linked worktree (`.git` is a file)
   `main` is checked out elsewhere and git will refuse; skip this step there.

```bash
test -f .git || { git checkout main && git pull; }
```
