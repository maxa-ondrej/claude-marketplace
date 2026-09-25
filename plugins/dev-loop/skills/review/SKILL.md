---
name: review
description: Review the current changes through two lenses — conventions/quality and adversarial logic/risk — and write the findings as checklists under .dev-loop/review/ for /implement to fix. Pass --pr to pull a PR's human review comments in as findings too.
argument-hint: [--pr] [files or focus]
---

# Review Skill

Find what is wrong with the code that was just written, and leave it on disk in a form
`/dev-loop:implement --fix` can consume.

**Outputs:** `.dev-loop/review/reviewer.md`, `.dev-loop/review/hater.md`, and with `--pr`,
`.dev-loop/review/pr-comments.md`

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

```bash
gh pr view --json number -q '.number'
gh api repos/{owner}/{repo}/pulls/{pr}/reviews
gh api repos/{owner}/{repo}/pulls/{pr}/comments
```

Invoke the `architect` agent to triage them into the same checklist format at
`.dev-loop/review/pr-comments.md`. It decides which are actionable and drops the ones that
contradict AGENTS.md or add complexity for its own sake — with a one-line reason for each drop, so
you can answer the reviewer.

Human comments outrank both agents. A person who left a comment gets an answer even when the
architect drops it.

### 4. Report

A count per severity per file, and the paths. Do not print the findings into chat — they are in the
files, and reprinting them is how a twenty-nit list buries the one blocker.

Then tell the user their options: `/dev-loop:implement --fix` to fix the must-fixes, or read the
files first.

### 5. Fix loop

If the user says fix, invoke `/dev-loop:implement --fix`. It ticks findings off and calls back here
to confirm. Repeat until no `Blocker` or `Must Fix` findings remain unchecked.
