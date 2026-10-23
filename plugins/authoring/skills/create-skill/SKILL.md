---
name: create-skill
description: Create a new Claude Code skill. Analyzes existing skills and agents, then scaffolds a new skill definition.
argument-hint: <skill-name> <description of what it does>
---

> **Talk to the user in ASD-STE100 Simplified Technical English.** Approved words only. One idea
> per short sentence. Active voice. The same word for the same thing every time. No jargon, no
> idioms, no long noun groups. This governs what you say, never what the code or the commit says.

# Create Skill

Create a new Claude Code skill by invoking the `skill-creator` agent.

## Execution

Invoke the `skill-creator` agent with: $ARGUMENTS

If no arguments provided, ask the user what skill they want to create.
