---
name: describe
description: Write or refresh the description of the current PR or MR from its commits, diff and plan. Shows the text and asks before it posts. Use when the user asks to write, update, or fix a PR/MR description, or when a description has gone stale after new commits.
argument-hint: [nothing — describes the current branch's PR] | [what to emphasise]
---

> **Talk to the user in ASD-STE100 Simplified Technical English.** Approved words only. One idea
> per short sentence. Active voice. The same word for the same thing every time. No jargon, no
> idioms, no long noun groups. This governs what you say, never what the code or the commit says.

# Describe Skill

Write the description a reviewer reads first. `/dev-loop:ship` calls this skill, so there is one
body format in one place. It also runs on its own, to refresh a description after more commits.

> **Never post a description the user has not seen.** Colleagues read it, and a wrong description
> sends a reviewer down the wrong path. Show it, then ask.

## Execution

### 1. Find the base and the PR

```bash
BASE=$(git symbolic-ref --short -q refs/remotes/origin/HEAD) || git remote set-head origin --auto
```

Never assume `origin/main` — this repo may use `master`. Then find the PR for the branch, per
`FORGE.md`. No PR yet? That is normal when `/dev-loop:ship` calls you before it opens one: write
the body and return it, do not try to post.

### 2. Read what changed

```bash
git log --reverse --format='%s%n%b' "$BASE"..HEAD
git diff --stat "$BASE"
git diff "$BASE"
```

**The diff decides what the description says.** Commit subjects say what the author meant to do;
the diff says what they did.

Then, if they exist, read `.dev-loop/plan.md` for why the work was done and
`.dev-loop/implementation.md` for where the code went away from the plan. Use them for the
reasoning only.

**Never describe something the diff does not contain.** A plan is a statement of intent, and parts
of it are often cut. If the plan and the diff do not agree, the diff wins, and the difference is
worth a line in the description.

### 3. Write the body

```markdown
## Summary
- <what this changes, and why — one bullet per point, 1 to 3 bullets>

## Changes
- `path/to/file.ts` — <what changed here>

## Test plan
- <how it was checked: the commands that ran, what they proved>
```

Rules:

- **Plain English, as short as possible.** Bullets when there is more than one point. Plain words
  over jargon. Never restate the diff line by line, and never pad a section to look thorough.
- An honestly empty section beats invented content. No tests? Say what you did instead.
- Leave out `## Changes` for a change of one or two files — the diff already shows it.
- Add anything the reviewer must know to review safely: a migration, a breaking change, a flag that
  must be set, an order the deploy must follow.
- If the project's tracker workflow wants a ticket link in the body, include it. Getting the link
  belongs to that tracker plugin, not here.
- `$ARGUMENTS` says what to emphasise. It never overrides the diff.

### 4. Show it and ask

Print the body in full. Then ask with `AskUserQuestion`: **Post** / **Edit** / **Cancel**.

On **Edit**, take their words, show the new body, and ask again. Post only what they approved.

Called by `/dev-loop:ship` before the PR exists? Return the approved body to it instead of posting.

### 5. Post

Update the description per `FORGE.md` (**Set the PR description**). Keep every part of the body the
user approved.

### 6. Report

One line: posted or returned, and the PR link.
