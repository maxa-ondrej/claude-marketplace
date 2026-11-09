---
name: tidy-history
description: Reshape the branch's commits into logical commits that each do one thing, reword them to say why, and force-push with a lease. Pass --round to fold only this review round's fix commits into one, which keeps reviewed commits untouched. Use when the user asks to clean up, squash, reshape, or rewrite the commit history of a branch.
argument-hint: [nothing — reshapes the whole branch] | --round | [how to group the commits]
---

> **Talk to the user in ASD-STE100 Simplified Technical English.** Approved words only. One idea
> per short sentence. Active voice. The same word for the same thing every time. No jargon, no
> idioms, no long noun groups. This governs what you say, never what the code or the commit says.

# Tidy History Skill

Turn `wip`, `fix test`, `fix test again`, `address review`, `typo` into commits a reviewer can read
one at a time.

> **The code must not change.** This skill moves the same work into different commits. Step 5
> proves the tree is identical and undoes everything if it is not. Never skip it.

Two modes:

- **Full reshape** (default) — regroup every commit on the branch.
- **Round mode** (`--round`) — fold only the commits added since the last push into one commit.
  `/dev-loop:ship` runs this after each review round.

## Round mode (`--round`)

A review round leaves commits like `address review` and `fix test` on top of commits a reviewer has
already read. Round mode squashes **only the new ones**, into one commit that says what the round
changed.

```bash
git fetch origin
git log --oneline @{upstream}..HEAD          # the round's commits
```

Nothing new, or one commit already? Say so and stop — there is nothing to fold.

Then soft-reset to the upstream commit and make one commit from the result:

```bash
git reset --soft @{upstream}
```

Name it for the round, not for the mechanics: `Address review: guard the empty batch`, never
`fixes`. Then run **step 5** of the full mode to prove the code did not change, and **step 6** to
push.

**Round mode does not touch any commit the reviewer has read, so review threads keep their
anchors.** That is the whole reason it exists, and the reason it is the automatic mode. Use the
full reshape by hand, when you accept that the threads lose their anchors.

## Execution (full reshape)

### 1. Refuse unless it is safe to start

```bash
git status --porcelain        # must be empty
BASE=$(git symbolic-ref --short -q refs/remotes/origin/HEAD) || git remote set-head origin --auto
git log --oneline "$BASE"..HEAD
```

Stop, and say why, when:

- **The working tree is dirty.** Commit or stash first. A rewrite with uncommitted work in the tree
  can lose it.
- **The branch is the default branch.** Never rewrite `main` or `master`.
- **There is one commit, or none.** Nothing to reshape.
- **A merge commit is in the range.** Reshaping across a merge needs a human.

### 2. Record the undo point

```bash
git rev-parse HEAD                              # write this SHA into your report
git branch "backup/$(git branch --show-current)" HEAD
```

The branch is the safety net. Keep it until the user says the result is good.

A `backup/<branch>` already there from an earlier run? Ask before you overwrite it.

### 3. Plan the grouping

Read every commit and the whole diff:

```bash
git log --reverse --format='%h %s%n%b' "$BASE"..HEAD
git diff "$BASE" --stat
git diff "$BASE"
```

Understand what each change does, and why, before you group anything.

Each commit has **one logical purpose**: a refactor, a new model or migration, one piece of
business logic, one API endpoint, a config change. Never `WIP`, never `fix typo`.

Order them so each builds on the last:

1. **Preparation** — refactors and infrastructure that later commits need
2. **Features** — the work itself
3. **Cleanup** — anything left over

Rules:

- **Keep tests in the same commit as the code they cover.** A commit that adds behaviour and a
  commit that tests it are one commit, not two.
- **Only the last commit has to pass.** A commit in the middle may fail to build or fail its
  tests. Do not split or merge commits to keep every one of them green — group them so a reviewer
  can read them, and let the branch tip be the thing that works.
- Squash the noise into the commit it belongs to: fixups, a revert and the thing it reverted, and
  formatting-only churn.
- Fewer, larger commits beat many tiny ones. Three good commits beat eight.
- `$ARGUMENTS` says how to group. Follow it.

**Show the plan and wait.** Print, for each planned commit, its subject line and the files or hunks
that go into it, plus which old commits it absorbs. Then ask with `AskUserQuestion`:
**Rewrite** / **Change the plan** / **Cancel**.

Nothing is rewritten until the user says so. On **Change the plan**, take their grouping, print the
new plan, and ask again.

### 4. Rewrite

`git rebase -i` cannot run here, so do it with a soft reset. The reset keeps every change in the
index; you then commit it in groups:

```bash
git reset --soft "$BASE"
git restore --staged .
```

Then, for each planned commit: stage exactly its paths (`git add <paths>`, or `git add -p` through
a patch file for a split inside one file) and commit with a HEREDOC message.

Messages follow the repo's own convention from `git log` on the base branch. Within that:

- **Subject: imperative, 72 characters or fewer.** "Add the retry window", not "Added" or "Adding".
- **Body: short, and about why.** Leave it out when the subject says everything.
- Simple technical English. No `Co-Authored-By`, `Generated-By`, or AI attribution footers.

### 5. Prove the code did not change

```bash
git diff "backup/<branch>" HEAD          # MUST print nothing
```

Output of any kind means the rewrite lost or changed work.

**Undo at once, do not try to repair it:**

```bash
git reset --hard "backup/<branch>"
```

Then tell the user what the difference was. A failed tidy that was undone is a good outcome. A
tidy that quietly changed the code is the worst thing this skill can do.

Then run the repo's typecheck and tests on the branch tip. The tree is identical to the backup, so
they must pass exactly as they did before. A failure here means the rewrite went wrong, not that
the code is wrong — undo and report.

Do not test the commits in the middle. They are allowed to fail.

### 6. Push

Check what the force-push would disturb first:

```bash
git fetch origin
git rev-list --count HEAD..@{upstream}           # commits on the remote you do not have
```

Not zero means somebody else pushed. Stop and tell the user — `--force-with-lease` would fail
anyway, and their commits need a human decision.

Warn the user, before you push, when the PR has review threads: those threads point at the old
commits, and the rewrite leaves their anchors behind. Say how many threads will be affected, from
`FORGE.md` (**Threads**).

Push without asking. The user approved the rewrite at the plan gate in step 3, and the lease is
the protection that matters here.

```bash
git push --force-with-lease
```

The lease makes the push fail, rather than overwrite, if the remote moved since your fetch. Never
use a plain `--force`.

### 7. Report

The old commit list, the new one, the backup branch name, and the check from step 5. Tell the user
to delete the backup branch when they are happy:

```bash
git branch -D "backup/<branch>"
```
