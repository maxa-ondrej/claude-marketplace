---
name: research
description: Investigate APIs, libraries, framework docs, and codebase patterns needed for a piece of work, and write the findings to .dev-loop/research.md. Use before planning anything unfamiliar, or when the user asks how something works or which API to use.
argument-hint: <what to research>
---

# Research Skill

Answer the open questions a piece of work depends on, and leave the answers on disk.

**Output:** `.dev-loop/research.md`

## Execution

### 1. Scope the questions

From `$ARGUMENTS`, list the concrete questions research must answer — an API's real signature, how
an existing feature does the same thing, which of two libraries the repo already has, what a
framework version supports.

If `$ARGUMENTS` is vague ("research the payments thing"), ask what specifically is unclear before
spending agents on it.

Skip this skill entirely when the work uses patterns already obvious in the codebase. Research that
confirms what everyone knows is waste.

### 2. Run the researchers

Invoke the `researcher` agent — **one per independent topic, in parallel** (all Agent calls in one
message). Each gets one topic and the context for why it matters.

Tell each researcher to append its section to `.dev-loop/research.md` under a `## <topic>` heading,
creating the file if it does not exist. Create `.dev-loop/` first and add it to `.gitignore` if it
is not already there.

### 3. Report

Summarize in three lines or less: what was answered, anything that stayed unknown, and the file
path. The detail lives in the file — do not re-print it into chat.

Flag any question that came back unanswered. An unanswered question is a risk `/plan` needs to see.
