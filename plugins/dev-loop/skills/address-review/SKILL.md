---
name: address-review
description: Address the review comments on the current PR or MR — triage them, fix them, ship, then go thread by thread letting you reply and resolve each one. Works on GitHub and GitLab. Use when the user asks to address, apply, or answer PR/MR review comments or feedback.
---

# Address Review Skill

Take the human comments on the current PR all the way to answered threads.

The triage, the fixing and the shipping already exist as skills — this one drives them and then does
the part none of them do: closing the loop with the people who commented.

> **Never post a reply or resolve a thread the user did not explicitly choose, thread by thread, in
> step 4.** Post only wording the user has seen. These are messages to colleagues under the user's
> name — there is no undo that unsends them.

## Execution

### 1. Triage

Detect the forge and find the PR, per `FORGE.md` — every command this skill runs comes from
there, so it works the same on GitHub and GitLab. No PR for the current branch, or no comments?
Say so and stop.

Then `/dev-loop:review --pr`. It pulls the comments, reads the repo's review lessons, triages into
`.dev-loop/review/pr-comments.md` as checkboxes, and flags anything that contradicts a lesson.

Show the user the apply/drop split before going further. They may overrule any of it.

### 2. Fix

`/dev-loop:implement --fix` — it builds the fixes and ticks the findings off.

Nothing to apply (every comment dropped or answered in words)? Skip to step 4.

### 3. Ship

`/dev-loop:ship`. Keep the commit SHAs — each reply cites the commit that addressed it.

### 4. Reply and resolve, one thread at a time

Review comments live on threads. Fetch them with their ids — **Threads** in `FORGE.md`.

Skip threads already resolved, and the ones that cannot be resolved at all (GitLab system notes).
Pair each remaining one with its triage outcome and commit SHA. Top-level PR comments are not
threads — answer those with the plain comment command at the end.

Then, **one thread at a time, in order**:

1. Print the thread: author, `path:line`, the comment, and its outcome — `Fixed in <sha>` or
   `Dropped: <reason>`.
2. Draft the reply you *would* send. Fixed → what changed and the SHA. Dropped → why, plainly, no
   defensiveness. Plain English, as short as stays accurate.
3. Ask with `AskUserQuestion`: **Reply & resolve** / **Reply only** / **Resolve only** / **Nothing**.
   They may pick Other to edit the wording — show the revised draft and treat their choice as
   applying to that.
4. Act on the answer before moving on, using **Reply into a thread** and **Resolve a thread** from
   `FORGE.md`. Reply first, then resolve, when they chose both.

Offer resolve only for a thread that can be resolved; for the rest the choice is reply or nothing.

A thread the user leaves untouched stays open. That is a valid answer, not a failure.

### 5. Report

Comments fixed with their SHAs, comments dropped with reasons, what happened per thread
(replied / resolved / both / left), any lessons saved in step 1, and the PR's CI state.
