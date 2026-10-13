---
name: address-review
description: Address the review comments on the current PR — triage them, fix them, ship, then go thread by thread letting you reply and resolve each one. Use when the user asks to address, apply, or answer PR review comments or feedback.
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

```bash
gh pr view --json number,url -q '.number, .url'
gh repo view --json owner,name -q '.owner.login, .name'
```

No PR for the current branch, or no comments? Say so and stop.

Substitute those three values literally into the commands below — each Bash call is a fresh shell,
so a variable set in one does not survive into the next.

Then `/dev-loop:review --pr`. It pulls the comments, reads the repo's review lessons, triages into
`.dev-loop/review/pr-comments.md` as checkboxes, and flags anything that contradicts a lesson.

Show the user the apply/drop split before going further. They may overrule any of it.

### 2. Fix

`/dev-loop:implement --fix` — it builds the fixes and ticks the findings off.

Nothing to apply (every comment dropped or answered in words)? Skip to step 4.

### 3. Ship

`/dev-loop:ship`. Keep the commit SHAs — each reply cites the commit that addressed it.

### 4. Reply and resolve, one thread at a time

Review comments live on threads. Fetch them with their thread IDs — resolving is GraphQL-only, so
there is no REST shortcut:

```bash
gh api graphql -f query='
  query($owner:String!,$repo:String!,$pr:Int!) {
    repository(owner:$owner, name:$repo) {
      pullRequest(number:$pr) {
        reviewThreads(first:100) { nodes {
          id isResolved isOutdated path line
          comments(first:1) { nodes { databaseId author { login } body } }
        } }
      }
    }
  }' -F owner=<owner> -F repo=<repo> -F pr=<number>
```

Skip threads already `isResolved`. Pair each remaining one with its triage outcome and commit SHA.
Top-level PR comments are not threads — answer those with `gh pr comment` at the end.

Then, **one thread at a time, in order**:

1. Print the thread: author, `path:line`, the comment, and its outcome — `Fixed in <sha>` or
   `Dropped: <reason>`.
2. Draft the reply you *would* send. Fixed → what changed and the SHA. Dropped → why, plainly, no
   defensiveness. Plain English, as short as stays accurate.
3. Ask with `AskUserQuestion`: **Reply & resolve** / **Reply only** / **Resolve only** / **Nothing**.
   They may pick Other to edit the wording — show the revised draft and treat their choice as
   applying to that.
4. Act on the answer before moving on:

```bash
# reply into the thread (databaseId of its first comment)
gh api "repos/<owner>/<repo>/pulls/<number>/comments/<databaseId>/replies" -f body='<the message the user approved>'

# resolve — only when the user chose to
gh api graphql -f query='mutation($t:ID!){ resolveReviewThread(input:{threadId:$t}){ thread { isResolved } } }' -f t=<thread-id>
```

A thread the user leaves untouched stays open. That is a valid answer, not a failure.

### 5. Report

Comments fixed with their SHAs, comments dropped with reasons, what happened per thread
(replied / resolved / both / left), any lessons saved in step 1, and the PR's CI state.
