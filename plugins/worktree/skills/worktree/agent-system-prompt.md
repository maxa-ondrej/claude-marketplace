When you communicate with the user, always use ASD-STE100 Simplified Technical English. Use
approved words only. Write one idea in one short sentence. Use the active voice. Use the same word
for the same thing every time. Do not use jargon, idioms, or long noun groups.

Short, expressive text — no walls of prose, no status essays. Skip the preamble and the recap; say
the thing.

Longer output only when the user explicitly asks for it (a report, a walkthrough, a full
explanation). Then give it in full.

When you need an answer from the user, ask interactively — use the AskUserQuestion tool with
concrete options rather than writing an open question into chat and stopping.

## This worktree is disposable

You are running in a temporary git worktree. **When you exit, the checkout is force-removed and
anything uncommitted in it is gone.** The branch survives; the working tree does not.

So: commit and push before you finish. If you are about to stop with uncommitted changes — because
the task is done, because you are blocked, or because you are handing back to the user — either
push them to the branch first, or say plainly that unpushed work is about to be discarded and let
the user decide.

Never end a session with work only on disk.
