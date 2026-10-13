---
name: implement
description: Build an approved plan — tests first, then code, one developer per package in parallel — or fix the findings /review wrote. Writes .dev-loop/implementation.md. Use whenever the user asks to implement, build, code, or fix.
argument-hint: [nothing — builds .dev-loop/plan.md] | --fix
---

# Implement Skill

Write the code. Two modes:

- **Build mode** (default) — build `.dev-loop/plan.md` from scratch.
- **Fix mode** (`--fix`) — work through the open findings in `.dev-loop/review/`.

**Output:** `.dev-loop/implementation.md`

## Build mode

### 1. Load the plan

Read `.dev-loop/plan.md` and `.dev-loop/design.md` if present. No plan on disk? Run
`/dev-loop:plan` first — do not improvise one. If `$ARGUMENTS` describes work that is not in the
plan, say so and stop.

### 2. Prepare shared contracts first

If the plan changes a shared types/schema/contracts package that other packages import, invoke one
`developer` agent for that package alone and have it build the package. Everything downstream
imports from it, so it has to exist before tests are written.

Skip if the plan touches no such package.

### 3. Tests first

Invoke the `tester` agent in **TDD mode** with the plan's test specification. It writes every test
file before any implementation. They fail. That is the point.

Then run the repo's format and typecheck commands (from AGENTS.md) yourself — a Bash call, not an
agent — and hand any failures back to the tester.

### 4. Implement

The lessons file binds whoever writes the code, not just whoever planned it. The `developer` and
`tester` agents read it themselves; if you edit directly from this thread instead of delegating,
read it first — most lessons are about things a plan never mentions.

One `developer` agent **per affected package or app, in parallel** (all Agent calls in one
message). Each receives the plan filtered to its own scope, the design spec if relevant, and the
note that tests already exist and must pass.

One package affected? One agent. Do not parallelize for its own sake.

### 5. Verify

Run the repo's format, codegen, typecheck, and test commands (from AGENTS.md). These are Bash
calls from this thread — there is no agent for running a command and reading its exit code.

On failure, hand the errors to the `developer` agent that owns those files, then re-run. Repeat
until green. No round limit.

Then invoke the `tester` agent in **verify mode** to fill coverage gaps the plan's spec missed.

### 6. Write the summary

Write `.dev-loop/implementation.md`: one section per task, the files created and modified, and any
deviation from the plan with its reason. Deviations matter more than the file list — `/review`
reads them.

### 7. Report

Tasks done, checks green, the file path. Three lines.

---

## Fix mode (`--fix`)

### 1. Read the findings

Read every file in `.dev-loop/review/`. Collect the unchecked (`- [ ]`) findings at **Blocker** and
**Must Fix** severity. Include **Should Fix** only if the user asked for them.

Nothing unchecked? Say so and stop.

### 2. Fix them

Group findings by the package that owns them and invoke one `developer` agent per group, in
parallel. Each gets its findings verbatim.

A finding that contradicts AGENTS.md is not fixed silently — the developer notes the conflict and
leaves it. Surface those to the user.

### 3. Tick them off

Change `- [ ]` to `- [x]` for each finding actually fixed, in the file it came from. Leave the rest
unchecked, with a one-line note on why.

### 4. Verify and append

Re-run format, typecheck, and tests. Append a `## Review fixes` section to
`.dev-loop/implementation.md` listing what changed.

Then re-run `/dev-loop:review` to confirm the fixes hold.
