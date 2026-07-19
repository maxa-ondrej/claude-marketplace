---
name: refactor
description: Refactor code with clear before/after explanation. Verifies changes with tests before completion. Use when the user asks to refactor, clean up, or restructure existing code.
argument-hint: <file> [refactoring-goal]
---

# Code Refactor

Refactor code, with a deeper design review if one is available.

## Execution

### Step 1: Pre-refactor review (optional)

If a `principal-engineer` agent is available, invoke it with the target files and refactoring goal. It identifies type safety issues, anti-patterns, architectural concerns, and suggests refactoring priorities. Skip this step if no such agent is available.

### Step 2: Refactor

Invoke the `refactorer` agent with the target files, the refactoring goal, and the pre-refactor findings (if produced).

### Step 3: Post-refactor review (optional)

If a `principal-engineer` agent is available, invoke it on the changed files to verify the refactoring improved type safety and correctness without introducing regressions.

If it reports **critical issues**, send them back to the `refactorer` agent and repeat Steps 2–3 until clean.

## Target

Refactor: $ARGUMENTS

If no target specified, ask the user which file to refactor and what improvement they want.
