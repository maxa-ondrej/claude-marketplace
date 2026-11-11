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
>
> **One exception: a thread the user wrote themselves.** Reply to it without asking — they are
> reading their own note, not receiving a message. Still never resolve it.

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
threads — answer those with the plain comment command at the end, footer included.

Find out who you are posting as — **Who you are posting as** in `FORGE.md` — and split the
threads by author.

**A thread the user wrote themselves is not a conversation with anybody.** It is their own note on
their own code, so there is nobody to ask and nobody to answer:

- **Post the draft reply.** Do not ask first. Do not offer Send, Edit or Skip.
- **Never resolve it.** The user closes their own notes, in their own time.
- Draft it to the same standard as any other reply, and show what you posted in the report.

Every other thread follows the steps below.

Then, for **every thread a colleague wrote**, one at a time, in order:

1. Print the thread: author, `path:line`, the comment, and its outcome — `Fixed in <sha>` or
   `Dropped: <reason>`.
2. Draft the reply. Fixed → what changed and the SHA. Dropped → why, plainly, no defensiveness.

   **Write it in simple English. Short.** One or two sentences is the normal length. Say what you
   did, not how you thought about it. No jargon the reviewer did not use, no hedging, no thanking
   them for the "great catch". If the draft runs past three sentences, cut it.

3. Ask with `AskUserQuestion` — **one question, these four options, every time**:

   - **Reply and resolve** — post the draft, then resolve the thread
   - **Reply only** — post the draft, leave the thread open
   - **Edit message** — they give new wording; show the new draft and ask again
   - **Do nothing** — leave the thread alone

   Reply first, then resolve, when they pick both. Post with **Reply into a thread** from
   `FORGE.md`, and resolve with **Resolve a thread**.

   Every reply ends with the footer from **Sign everything you post** — show it in the draft, so
   what they approve is what goes up.

   The one time the options change: a thread that cannot be resolved, such as a GitLab system note.
   Drop **Reply and resolve** there and say why.

A thread left open is a valid answer, not a failure.

For the user's own threads, nothing is asked at all: the draft is posted and the thread is left
open.

### 5. Report

Comments fixed with their SHAs, comments dropped with reasons, what happened per thread
(replied / resolved / both / left), any lessons saved in step 1, and the PR's CI state.
