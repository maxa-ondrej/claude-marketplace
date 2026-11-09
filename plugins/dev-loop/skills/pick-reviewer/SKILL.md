---
name: pick-reviewer
description: Propose who should review the current PR or MR, ranked from code owners, review history, file history and current review load, then assign the ones the user picks. Use when the user asks who should review this, to pick or select a reviewer, or to assign reviewers.
argument-hint: [nothing — uses the current PR] | [a name to include or exclude]
---

> **Talk to the user in ASD-STE100 Simplified Technical English.** Approved words only. One idea
> per short sentence. Active voice. The same word for the same thing every time. No jargon, no
> idioms, no long noun groups. This governs what you say, never what the code or the commit says.

# Pick Reviewer Skill

Say who should review this change, and why each person is on the list.

> **Never assign a reviewer the user did not choose.** Assigning gives a colleague work. The user
> picks; this skill only proposes.

## Execution

### 1. Find the PR and what it changes

Find the PR per `FORGE.md`. No PR for the branch? Say so and stop.

```bash
BASE=$(git symbolic-ref --short -q refs/remotes/origin/HEAD) || git remote set-head origin --auto
git diff --name-only "$BASE"
```

### 2. Read the repo's own rule first

Does the repo document who reviews what — a level model, a rota, a rule that one team owns one
directory? Look in `AGENTS.md`, `CLAUDE.md`, `CLAUDE.local.md` and `CONTRIBUTING.md`.

**A documented rule beats every signal below.** Follow it, and use the signals only to fill in the
names it does not give. Say which rule you followed.

### 3. Collect the signals

Run these together, in one message — see **Reviewer signals** in `FORGE.md`:

1. **Code owners** for the changed files. The strongest signal: these people are responsible for
   this code.
2. **Who reviews your work most**, from your own merged PRs. These are the people who really
   review for you.
3. **Who last changed these files** — real context on this code, even with no `CODEOWNERS` entry:
   ```bash
   git log --format='%an' -n 50 -- <changed paths> | sort | uniq -c | sort -rn
   ```
4. **How much review work each candidate already has** — count the open PRs waiting for them.

Remove the author from every list. Remove bots unless a bot is the documented reviewer.

### 4. Rank them

Order by the signals, in this order of weight:

1. A code owner for these files.
2. Then, a person who reviews your work often **and** has context on these files.
3. Then, either one alone.

Use the review load last, to choose between people the first three rules rate the same. A person
with ten open reviews is a slower review than a person with one, so prefer the lighter queue when
nothing else separates them.

Show each candidate with the reason, so the user can argue with it:

```
Petr Morávek     code owner (src/billing) · reviews 12 of your last 30 · 3 open reviews
Ondřej Novák     wrote 8 of the last 10 commits here · reviews 6 of your last 30 · 9 open reviews
Jakub Podlesný   code owner (src/billing) · 1 open review
```

Say plainly when a signal is missing — no `CODEOWNERS` file, a new file nobody has touched, an
empty review history. A short honest list beats a padded one.

### 5. Let the user choose

Ask with `AskUserQuestion`, `multiSelect: true`, with the ranked candidates as options. Keep the
**Other** choice, so the user can name somebody the signals did not find.

`$ARGUMENTS` can name a person to include or to leave out. Honour it.

### 6. Assign

Assign only the chosen people, per `FORGE.md` (**Assign reviewers**). Then say who was assigned,
and name anybody the forge refused — a user with no access to the repo, for example.
