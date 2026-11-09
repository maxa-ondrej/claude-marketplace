---
name: rebase
description: Stash the working tree, rebase the branch onto the default branch, resolve the conflicts, force-push with a lease, and pop the stash. Use when the user asks to rebase, to update a branch onto main or master, or to resolve rebase conflicts.
argument-hint: [nothing — rebases onto the default branch] | [a base ref]
---

> **Talk to the user in ASD-STE100 Simplified Technical English.** Approved words only. One idea
> per short sentence. Active voice. The same word for the same thing every time. No jargon, no
> idioms, no long noun groups. This governs what you say, never what the code or the commit says.

# Rebase Skill

Move the branch onto current `main` or `master`, deal with the conflicts, and put the working tree
back as it was.

> **A wrong conflict resolution loses code silently.** It compiles, the tests may even pass, and the
> lost line only turns up weeks later. Step 4 says which conflicts you may resolve and which ones
> you must hand back.

## Execution

### 1. Find the base, and make a safety net

```bash
BASE=$(git symbolic-ref --short -q refs/remotes/origin/HEAD) || git remote set-head origin --auto
git fetch origin
git branch "backup/$(git branch --show-current)" HEAD
git rev-parse HEAD              # keep this SHA for the report
```

**Never assume `origin/master` or `origin/main`** — the repo decides which it is, and rebasing onto
the wrong one rewrites the branch against the wrong history. A ref in `$ARGUMENTS` wins over the
detected one.

Stop when the branch is the default branch. Never rebase `main` or `master` itself.

Already up to date (`git rev-list --count HEAD..$BASE` is 0)? Say so and stop. There is nothing to
rebase, and a pointless force-push still breaks review-thread anchors.

### 2. Stash the working tree

```bash
git stash push --include-untracked -m "rebase-$(git branch --show-current)"
git stash list | head -1         # confirm it is there
```

Nothing to stash? Skip this and step 6. Say which you did.

**Confirm the stash exists before you rebase.** A rebase over a dirty tree can lose the uncommitted
work, and `git stash push` writes nothing when the tree is clean.

### 3. Rebase

```bash
git rebase "$BASE"
```

Clean? Go to step 5.

### 4. Resolve the conflicts

For each conflicted file, read **both** sides and the surrounding code before you touch it:

```bash
git status --short | grep '^UU\|^AA\|^DU\|^UD'
git log --oneline --left-right "$BASE"...HEAD      # what each side did
git diff                                           # the conflict markers
```

**Resolve, when you can see what both sides meant and the answer keeps both intentions:**

- Both sides added a line to the same list, import block, or switch — keep both, in a sensible order.
- One side renamed a symbol the other side used — apply the rename to the other side's code.
- One side moved code the other side changed — move it, and carry the change with it.
- A lock file, a generated file, or a snapshot — regenerate it with the repo's own command rather
  than merging it by hand.

**Stop and hand it back, when the answer is a judgment call:**

- Both sides changed the same logic in different directions.
- A test and the code it covers disagree about what the behaviour should be.
- Keeping both sides would run the same work twice, or skip a guard.
- You cannot tell what one side was trying to do.

To hand back, leave the rebase in progress, print the file, both sides, and what you think each was
for. Then let the user decide. `git rebase --abort` puts the branch back as it was, and the backup
branch is there either way.

**Never resolve with `--ours` or `--theirs` across a whole file** unless the file is generated. It
throws away one side without reading it, which is the exact failure this skill exists to avoid.

Then continue:

```bash
git add <resolved paths>
git rebase --continue
```

Repeat for each conflicted commit.

### 5. Prove the result works

```bash
git diff "$BASE"...HEAD --stat           # the branch's own change, against the new base
```

Run the repo's format, typecheck, and test commands — or its own verification agents when it has
them, see `/dev-loop:implement`. These must pass. The base moved, so a failure here is real, not a
rewrite artefact: something on the base broke this branch, or a conflict went in wrong.

On failure: fix it when the cause is plain, or reset to the backup and report. Never push a branch
whose tests fail because of a conflict you resolved.

### 6. Push, then restore the working tree

Warn, before pushing, how many review threads will lose their anchor — `FORGE.md` (**Threads**).
A rebase rewrites every commit on the branch.

```bash
git push --force-with-lease
git stash pop
```

The lease refuses the push if the remote moved since the fetch in step 1. Never a plain `--force`.

**Pop after the push, never before.** If the pop conflicts, the pushed branch is still the clean
rebase, and the mess stays local.

Pop conflicts? Resolve it with the same rules as step 4, or leave it in the stash and say so:
`git stash list` keeps it, and nothing is lost.

### 7. Report

The base it rebased onto, how many commits moved, which conflicts you resolved and how, anything
you handed back, the test result, and the backup branch name:

```bash
git branch -D "backup/<branch>"        # when the user is happy
```
