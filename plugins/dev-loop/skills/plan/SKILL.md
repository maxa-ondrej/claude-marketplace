---
name: plan
description: Design an implementation plan for a story, feature, or bug — architecture, test spec, and UI mockups for any user-facing change — critique it, and render it as a visual page for the user to review and approve. Use whenever work needs designing before it gets built.
argument-hint: <story/bug description and task list>
---

# Plan Skill

Turn a description into an approved, reviewable plan. Writes no production code.

**Outputs:** `.dev-loop/plan.md`, `.dev-loop/design.md`*, `.dev-loop/mockups.html`*, and a published
visual artifact. (* only when there are user-facing changes.)

## Execution

### 1. Read what is already known

If `.dev-loop/research.md` exists, read it and pass it to the agents below. If the work depends on
something unfamiliar and no research file exists, run `/dev-loop:research` first.

### 2. Decide whether there is a UI surface

Is anything in `$ARGUMENTS` user-facing — a screen, a component, a CLI output, a chat command, an
email, an API response people read? If not, skip the designer entirely. A backend refactor does not
need mockups.

### 3. Draft (parallel)

Launch in **one message**:

1. **`architect`** — with the description, task list, and research findings. Writes
   `.dev-loop/plan.md`: files to touch, patterns to follow, order of operations, edge cases, risks,
   and the test specification `/dev-loop:implement` will build from.

2. **`designer`** *(only if step 2 said yes)* — with the description and the user-facing surfaces.
   Writes `.dev-loop/design.md` (user flow, component hierarchy, states, copy) **and**
   `.dev-loop/mockups.html` — static, self-contained HTML mockups of each screen or state, using
   the repo's real design tokens where they exist. Mockups are for judging the design, not for
   shipping: no framework, no build step, no imports.

### 4. Critique

Invoke the `hater` agent on `.dev-loop/plan.md` and `.dev-loop/design.md` together. It writes its
critique to `.dev-loop/review/plan-critique.md`.

If the verdict is `BLOCK` or there are blockers, send the critique back to the architect and/or
designer, have them revise their files in place, and re-run the hater. Repeat until no blockers
remain. Do not present a blocked plan to the user.

### 5. Render it for review

Build `.dev-loop/plan.html` from `plan.md` + `design.md` — a single page the user can read top to
bottom:

- The problem in two sentences
- Tasks as a numbered list, each with the files it touches
- A diagram when the change has a shape worth seeing (data flow, state machine, sequence) — load
  the `artifact-diagramming` skill rather than guessing at SVG
- The mockups inline, or linked as a supporting file
- Risks and open questions last, where they cannot be skimmed past

Load the `artifact-design` skill before writing the page. Publish it with the `Artifact` tool,
passing `mockups.html` as a supporting file when it exists. Hand the user the link.

If the `Artifact` tool is unavailable, render the plan as formatted markdown in chat instead and
give the file paths.

### 6. Get approval

Ask the user, interactively, to choose:

- **Accept** — the plan is approved, proceed
- **Change** — take their feedback to the architect/designer, revise, re-render, ask again
- **Decline** — stop

Never proceed to implementation on an unapproved plan.

### 7. Report

The artifact link, the file paths, and the number of tasks. Nothing else — the plan is in the plan.
