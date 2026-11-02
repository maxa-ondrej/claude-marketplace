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

Lessons do not live in `.dev-loop/`. They live in a Notion database, because they outlive the
branch and apply to every clone. `LESSONS.md` in this plugin has the commands and the rules.

`architect` reads them when it plans. `developer` and `tester` read them when they write.
`reviewer` applies them and marks each finding that comes from one. `/review --pr` is the only
writer, and only with the user's approval for each lesson.
