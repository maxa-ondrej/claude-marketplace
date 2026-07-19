---
name: revise
description: Fetch and implement code review comments on the current PR. Fixes issues, re-runs checks, commits, and pushes. Use when the user asks to address, revise, or fix review comments or PR feedback.
---

# Revise Skill

Address code review comments on the current PR.

## Execution

Follow these steps **in order**. Stop and report if any step fails.

---

### Step 1: Fetch review comments

Get the current PR number and fetch all review comments:

```bash
gh pr view --json number -q '.number'
gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments
```

If there are no comments, tell the user and stop.

---

### Step 2: Triage comments

Invoke the `architect` agent — and the `principal-engineer` agent in parallel, if one is available — with all fetched review comments and the current diff (`git diff main`).

The architect must:
1. Read each comment and understand what it asks for
2. Decide which comments are **relevant** — actionable feedback that improves the code
3. Skip comments that contradict AGENTS.md conventions, add unnecessary complexity, or are purely cosmetic/subjective
4. Return a list of relevant comments with a brief implementation note for each

The principal engineer (if invoked) must:
1. Review the same comments through the lens of type safety, correctness, and architectural impact
2. Flag any comments that point to deeper issues the architect may have missed (cascade impact, requirement leakage, anti-patterns)
3. Add additional implementation notes for comments that require deeper type-level or architectural fixes

Merge both outputs into the final triage list before proceeding.

---

### Step 3: Implement fixes

Invoke the `implement` skill with the architect's list of relevant comments as the task description. Each relevant comment becomes a task to address.

---

### Step 4: Ship

Invoke the `ship` skill to commit, push, and verify CI passes.
