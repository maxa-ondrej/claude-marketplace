---
name: create-skill
description: Create a new Claude Code skill. Analyzes existing skills and agents, then scaffolds a new skill definition.
argument-hint: <skill-name> <description of what it does>
---

# Create Skill

Create a new Claude Code skill by invoking the `skill-creator` agent.

## Execution

Invoke the `skill-creator` agent with: $ARGUMENTS

If no arguments provided, ask the user what skill they want to create.
