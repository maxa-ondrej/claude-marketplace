---
name: create-agent
description: Create a new Claude Code agent definition. Analyzes existing agents, then scaffolds a new agent with appropriate tools, model, and instructions.
argument-hint: <agent-name> <description of what it does>
---

# Create Agent

Create a new Claude Code agent by invoking the `agent-creator` agent.

## Execution

Invoke the `agent-creator` agent with: $ARGUMENTS

If no arguments provided, ask the user what agent they want to create.
