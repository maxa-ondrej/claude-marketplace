# dev-loop artifacts

Every skill in this plugin reads and writes markdown under `.dev-loop/` at the repo root.
`/work` runs the whole chain; each step is also runnable on its own, because the handoff is a file
rather than a conversation. That
directory is the handoff between skills — `/plan` reads what `/research` wrote, `/implement` reads
the plan, `/review` writes findings that `/implement` fixes.

```
.dev-loop/
  research.md          # /research  — findings, one section per topic
  plan.md              # /plan      — implementation plan + test spec (source of truth)
  design.md            # /plan      — UX spec, only when there are user-facing changes
  mockups.html         # /plan      — static UI mockups, only when there are user-facing changes
  plan.html            # /plan      — the reviewable render of plan.md + design.md
  implementation.md    # /implement — what was built, per task
  review/
    reviewer.md        # /review    — convention + quality findings
    hater.md           # /review    — adversarial logic + risk findings
```

One `.dev-loop/` per branch. In the worktree plugin's flow that means one per ticket, which is
exactly right — the directory dies with the worktree.

Add `.dev-loop/` to `.gitignore` on first use. These are working notes, not repo content.

Findings files use a checkbox per finding so `/implement` can tick them off and `/review` can see
what is left:

```markdown
- [ ] **Must Fix** `src/foo.ts:42` — loop assumes a non-empty list; throws on `[]`
      → Fix: guard with an early return
```

## Review lessons

Durable reviewer preferences — the rules a reviewer should never have to raise twice — live outside
`.dev-loop/`, because they outlive the branch:

```bash
"$(git rev-parse --path-format=absolute --git-common-dir)/dev-loop/review-lessons.md"
```

`--git-common-dir` resolves to the **main** repo's `.git` even from inside a worktree, so every
worktree of a repo reads and writes the same file, and it is never committed. One file per
repository.

`architect` reads it while planning, `developer` and `tester` while writing — most lessons are about
things a plan never mentions — and `reviewer` enforces it, tagging any finding that came from one.
`/review --pr` is the only thing that writes it, and only with per-lesson user approval.

```markdown
# Review Lessons

## <Category — Naming / Tests / Error handling / Architecture / Performance>
- <lesson, imperative> _(reviewer: <name>, <YYYY-MM-DD>)_
```
