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
`.dev-loop/review/pr-comments.md` plus any approved lessons added to the Notion lessons database
(see `LESSONS.md`).

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

Read the lessons **before** you make one triage decision — not at step 3b, because by then the
change that disagrees with a lesson is already written. `LESSONS.md` in this plugin has the
commands.

Read them in this thread, not only through the architect: you tell the user what to apply and what
to drop. A comment that **repeats** a recorded lesson shows the lesson is still correct — apply it.
A comment, or a drop you want to recommend, that **disagrees** with a lesson gets a flag and the
lesson quoted, before you recommend it. The user can still decide against the lesson. They decide
with the rule in view.

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

Read the lessons that are already there (`LESSONS.md` has the commands) and mark each candidate
**new**, **merge** (it makes a row that exists better), or **already there** (drop it).

Then ask the user with `AskUserQuestion` which ones to save. **Never write a lesson the user did
not approve, in words the user did not see.**

Add each approved lesson as one row. For a merge, change the row that exists — never add a row
that is almost the same.

### 4. Report

A count per severity per file, and the paths. Do not print the findings into chat — they are in the
files, and reprinting them is how a twenty-nit list buries the one blocker.

Then tell the user their options: `/dev-loop:implement --fix` to fix the must-fixes, or read the
files first. With `--pr`, also name the lessons saved — or say none generalized.

### 5. Fix loop

If the user says fix, invoke `/dev-loop:implement --fix`. It ticks findings off and calls back here
to confirm. Repeat until no `Blocker` or `Must Fix` findings remain unchecked.
