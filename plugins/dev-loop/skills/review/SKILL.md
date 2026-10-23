---
name: review
description: Review the current changes through two lenses — conventions/quality and adversarial logic/risk — and write the findings as checklists under .dev-loop/review/ for /implement to fix. Pass --pr to pull a PR's human review comments in as findings too.
argument-hint: [--pr] [files or focus]
---

> **Talk to the user in ASD-STE100 Simplified Technical English.** Approved words only. One idea
> per short sentence. Active voice. The same word for the same thing every time. No jargon, no
> idioms, no long noun groups. This governs what you say, never what the code or the commit says.

# Review Skill

Find what is wrong with the code that was just written, and leave it on disk in a form
`/dev-loop:implement --fix` can consume.

**Outputs:** `.dev-loop/review/reviewer.md`, `.dev-loop/review/hater.md`, and with `--pr`,
`.dev-loop/review/pr-comments.md` plus any approved lessons appended to the repo's review-lessons
file (see `ARTIFACTS.md`).

## Execution

### 1. Establish the diff

```bash
git diff --name-only main
git diff main
```

Nothing changed? Say so and stop.

If `.dev-loop/implementation.md` exists, read it — the deviations section tells both reviewers
where the code drifted from the plan, which is where the bugs live.

### 2. Review (parallel)

Launch in **one message**:

1. **`reviewer`** — conventions, type safety, security, performance, test adequacy. Writes
   `.dev-loop/review/reviewer.md`.
2. **`hater`** — logic, correctness, edge cases, design, risk. Writes `.dev-loop/review/hater.md`.

They overlap by design and disagree usefully. Do not merge them into one agent.

Both write findings as a checklist, one line per finding, so `/implement --fix` can tick them:

```markdown
- [ ] **Must Fix** `src/foo.ts:42` — loop assumes a non-empty list; throws on `[]`
      → Fix: guard with an early return
```

Severity is one of `Blocker`, `Must Fix`, `Should Fix`, `Nit`.

### 3. Pull in PR comments (`--pr` only)

Find the PR and read its comments per `FORGE.md` — **The PR for the current branch** and **Read
review comments**. GitHub and GitLab both work; the file says which commands to use.

Read the lessons file **before** you draft a single triage decision — not at step 3b, by which
point a contradicting change is already written:

```bash
cat "$(git rev-parse --path-format=absolute --git-common-dir)/dev-loop/review-lessons.md" 2>/dev/null
```

Read it in this thread, not only via the architect: you are the one who tells the user what to
apply and what to drop. A comment **restating** a recorded lesson confirms it still matters — apply
it without debate. A comment or a drop you are about to recommend that **contradicts** one gets
flagged with the lesson quoted, before you recommend anything. The user may still override it;
they do it knowing the rule exists.

Invoke the `architect` agent to triage them into the same checklist format at
`.dev-loop/review/pr-comments.md`. It decides which are actionable and drops the ones that
contradict AGENTS.md or add complexity for its own sake — with a one-line reason for each drop, so
you can answer the reviewer.

Human comments outrank both agents. A person who left a comment gets an answer even when the
architect drops it.

### 3b. Distill durable lessons (`--pr` only)

Some comments are one-off ("typo here"). Others state a standing preference the reviewer will raise
again next PR — a naming rule, a testing expectation, an error-handling pattern. Record those so
`architect` plans around them and `reviewer` enforces them.

A candidate lesson must be reusable across PRs, phrased as an imperative rule rather than a diff
("prefer the repository finder over hand-written queries for single-row lookups", not "change line
42"), and free of secrets, ticket IDs, and verbatim comment text. If nothing generalizes, skip this
step silently.

Read the existing file and mark each candidate **new**, **merge** (refines an entry already there),
or **already covered** (drop it):

```bash
LESSONS="$(git rev-parse --path-format=absolute --git-common-dir)/dev-loop/review-lessons.md"
cat "$LESSONS" 2>/dev/null
```

`--git-common-dir` resolves to the main repo's `.git` even from a worktree, so every worktree of
this repo shares one file and it is never committed.

Then ask the user, via `AskUserQuestion`, which to save. **Never write a lesson the user has not
approved in the wording they saw.** Approved ones go under their category heading — rewrite the
existing bullet in place for a merge, never duplicate it:

```markdown
# Review Lessons

## Tests
- Cover the error path, not just the happy path, for every new failure mode _(reviewer: <name>, <date>)_
```

Create the directory only when there is something to write (`mkdir -p "$(dirname "$LESSONS")"`), and
date entries with `date +%F`.

### 4. Report

A count per severity per file, and the paths. Do not print the findings into chat — they are in the
files, and reprinting them is how a twenty-nit list buries the one blocker.

Then tell the user their options: `/dev-loop:implement --fix` to fix the must-fixes, or read the
files first. With `--pr`, also name the lessons saved — or say none generalized.

### 5. Fix loop

If the user says fix, invoke `/dev-loop:implement --fix`. It ticks findings off and calls back here
to confirm. Repeat until no `Blocker` or `Must Fix` findings remain unchecked.
