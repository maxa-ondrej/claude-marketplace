---
name: tidy-history
description: Reshape the branch's commits into logical commits that each do one thing, reword them to say why, and force-push with a lease. Use when the user asks to clean up, squash, reshape, or rewrite the commit history of a branch.
argument-hint: [nothing — tidies the current branch] | [how to group the commits]
---

> **Talk to the user in ASD-STE100 Simplified Technical English.** Approved words only. One idea
> per short sentence. Active voice. The same word for the same thing every time. No jargon, no
> idioms, no long noun groups. This governs what you say, never what the code or the commit says.

# Tidy History Skill

Turn `wip`, `fix test`, `fix test again`, `address review`, `typo` into commits a reviewer can read
one at a time.

> **The code must not change.** This skill moves the same work into different commits. Step 5
> proves the tree is identical and undoes everything if it is not. Never skip it.

## Execution

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
git branch "tidy-backup-$(git rev-parse --short HEAD)" HEAD
```

The branch is the safety net. Keep it until the user says the result is good.

### 3. Plan the grouping

Read every commit and the whole diff:

```bash
git log --reverse --format='%h %s%n%b' "$BASE"..HEAD
git diff "$BASE" --stat
git diff "$BASE"
```

Group the changes so each commit does **one thing** and builds on the one before it. A good
grouping follows the shape of the work: the shared type or schema first, then the code that uses
it, then the tests, then anything that only touches style.

Rules:

- A commit that only repairs an earlier commit in this branch (`fix test`, `typo`,
  `address review`) is not a commit. Its changes belong in the commit it repairs.
- Never split a commit so that a middle commit does not build. Each commit must compile on its own.
- Fewer, larger commits beat many tiny ones. Three good commits beat eight.
- `$ARGUMENTS` says how to group. Follow it.

Show the user the plan — the old list, the new list, and which old commit goes where — before you
touch anything.

### 4. Rewrite

`git rebase -i` cannot run here, so do it with a soft reset. The reset keeps every change in the
index; you then commit it in groups:

```bash
git reset --soft "$BASE"
git restore --staged .
```

Then, for each planned commit: stage exactly its paths (`git add <paths>`, or `git add -p` through
a patch file for a split inside one file) and commit with a HEREDOC message.

Messages follow the repo's own convention from `git log` on the base branch. **Plain English, as
short as possible.** The subject says what the commit does. The body says **why**, only when why is
not obvious. No `Co-Authored-By`, `Generated-By`, or AI attribution footers.

### 5. Prove the code did not change

```bash
git diff "tidy-backup-<sha>" HEAD        # MUST print nothing
```

Output of any kind means the rewrite lost or changed work.

**Undo at once, do not try to repair it:**

```bash
git reset --hard "tidy-backup-<sha>"
```

Then tell the user what the difference was. A failed tidy that was undone is a good outcome. A
tidy that quietly changed the code is the worst thing this skill can do.

Also run the repo's typecheck and tests on the final commit. The tree is identical, so they must
pass exactly as they did before.

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

```bash
git push --force-with-lease
```

The lease makes the push fail, rather than overwrite, if the remote moved since your fetch. Never
use a plain `--force`.

### 7. Report

The old commit list, the new one, the backup branch name, and the check from step 5. Tell the user
to delete the backup branch when they are happy:

```bash
git branch -D "tidy-backup-<sha>"
```
