# Agent Configuration Patterns

This document shows how the agent crew is configured — the roles, model assignments, and coordination mechanisms that make the system work.

## Agent Definitions

Each agent is defined by four things: its model, its role description, the tools it can access, and the label it watches.

### Dotti — Data & Reports Agent

```yaml
name: Dotti
model: claude-haiku
role: |
  You are Dotti, a data and reports specialist. You handle:
  - Running queries against Neo4j and other databases
  - Formatting data into tables, summaries, and reports
  - Administrative tasks (status updates, label management)
  - Simple data transformations

  If a task requires writing code, complex reasoning, or
  architecture decisions, escalate to Finn.
label: dotti
tools:
  - linear (issue management)
  - neo4j (knowledge graph queries)
  - fibery (business data)
```

### Finn — Implementation Agent

```yaml
name: Finn
model: claude-sonnet
role: |
  You are Finn, a coding and implementation specialist. You handle:
  - Writing and modifying code (any language)
  - Bug fixes and feature implementation
  - Script creation and automation
  - Code review and refactoring

  If a task requires architecture decisions, complex system
  debugging, or design tradeoffs, escalate to Sage.
  If a task is just data queries or report formatting,
  de-escalate to Dotti.
label: finn
tools:
  - linear (issue management)
  - neo4j (knowledge graph)
  - github (code repositories)
  - bash (command execution)
  - file operations (read, write, edit)
```

### Sage — Architecture Agent

```yaml
name: Sage
model: claude-opus
role: |
  You are Sage, the tech lead and architecture specialist. You handle:
  - Architecture decisions and system design
  - Complex debugging across multiple systems
  - Technology evaluation and selection
  - Performance analysis and optimisation

  After resolving the complex part of a task, de-escalate
  routine follow-up work to Dotti or Finn.
label: sage
tools:
  - linear (issue management)
  - neo4j (knowledge graph)
  - github (code repositories)
  - bash (command execution)
  - file operations (read, write, edit)
  - web search (research)
```

## Escalation Protocol

The handoff format is standardised so receiving agents can parse context reliably:

```
[ESCALATION: <from> -> <to>]

Completed: <what was accomplished before escalating>

Blocked on: <specific reason for escalation>

Attempted: <what approaches were tried>

Context: <relevant data, file paths, query results>
```

### De-escalation Format

```
[DE-ESCALATION: <from> -> <to>]

Completed: <what the complex part achieved>

Remaining: <routine tasks for the receiving agent>
```

## Assignment Heuristics

Nova (the interactive advisor) uses these heuristics when creating issues:

| Task Type | Assign To | Rationale |
|-----------|-----------|-----------|
| Run a query, fetch data | Dotti | Structured I/O, no reasoning needed |
| Format a report, summarise | Dotti | Pattern-based output |
| Write or modify code | Finn | Requires code generation |
| Fix a bug | Finn | Requires code understanding |
| Build a new feature | Finn | Requires implementation |
| Design a system | Sage | Requires architectural reasoning |
| Debug a multi-service issue | Sage | Requires cross-system understanding |
| Evaluate a technology | Sage | Requires tradeoff analysis |

## Execution Commands

```powershell
# Process all agents in sequence
crew

# Run individual agents
dotti    # Process all issues labelled "dotti"
finn     # Process all issues labelled "finn"
sage     # Process all issues labelled "sage"

# View escalation metrics
crew-metrics
```

Each command:
1. Queries Linear for issues with the agent's label
2. Launches Claude Code CLI with the agent's model and role
3. Processes each issue autonomously
4. Updates Linear (comments, status, label changes)

## MCP Tool Access

Agents interact with external services through the Model Context Protocol (MCP). This provides a standardised tool interface without custom API code:

| MCP Server | Capabilities |
|------------|-------------|
| **Linear** | Create/update issues, manage labels, query projects |
| **Neo4j** | Run Cypher queries, create nodes, traverse relationships |
| **Fibery** | Query business data, update records |
| **Zotero** | Search research papers, retrieve citations |

MCP means adding a new service to the agent's toolkit is configuration, not code — register the MCP server and the agent can use it immediately.
