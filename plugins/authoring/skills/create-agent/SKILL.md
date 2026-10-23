---
name: create-agent
description: Create a new Claude Code agent definition. Analyzes existing agents, then scaffolds a new agent with appropriate tools, model, and instructions.
argument-hint: <agent-name> <description of what it does>
---

> **Talk to the user in ASD-STE100 Simplified Technical English.** Approved words only. One idea
> per short sentence. Active voice. The same word for the same thing every time. No jargon, no
> idioms, no long noun groups. This governs what you say, never what the code or the commit says.

# Create Agent

Create a new Claude Code agent by invoking the `agent-creator` agent.

## Execution

Invoke the `agent-creator` agent with: $ARGUMENTS

If no arguments provided, ask the user what agent they want to create.
