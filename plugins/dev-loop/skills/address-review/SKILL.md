---
name: address-review
description: Address the review comments on the current PR or MR — triage them, fix them, ship, then go thread by thread letting you reply and resolve each one. Works on GitHub and GitLab. Use when the user asks to address, apply, or answer PR/MR review comments or feedback.
---

> **Talk to the user in ASD-STE100 Simplified Technical English.** Approved words only. One idea
> per short sentence. Active voice. The same word for the same thing every time. No jargon, no
> idioms, no long noun groups. This governs what you say, never what the code or the commit says.

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

Skip threads already resolved, and GitLab system notes — those are not comments.
Pair each remaining one with its triage outcome and commit SHA. Top-level PR comments are not
threads — answer those with the plain comment command at the end.

Then, **one thread at a time, in order**:

1. Print the thread: author, `path:line`, the comment, and its outcome — `Fixed in <sha>` or
   `Dropped: <reason>`.
2. Draft the reply. Fixed → what changed and the SHA. Dropped → why, plainly, no defensiveness.

   **Write it in simple English. Short.** One or two sentences is the normal length. Say what you
   did, not how you thought about it. No jargon the reviewer did not use, no hedging, no thanking
   them for the "great catch". If the draft runs past three sentences, cut it.

3. Ask with `AskUserQuestion` whether to send it: **Send** / **Edit** / **Skip**. On Edit, take
   their wording, show it back, and ask again. Post only what they approved, with **Reply into a
   thread** from `FORGE.md`.
4. **Then always ask, separately: resolve this thread too?** Every thread, whether you replied or
   skipped — never bundle it into the reply question, and never resolve because replying seemed to
   settle it. Yes → **Resolve a thread** in `FORGE.md`. No → leave it open.

   Skip only this question for a thread that cannot be resolved (a GitLab system note), and say why.

A thread left open is a valid answer, not a failure.

### 5. Report

Comments fixed with their SHAs, comments dropped with reasons, what happened per thread
(replied / resolved / both / left), any lessons saved in step 1, and the PR's CI state.
