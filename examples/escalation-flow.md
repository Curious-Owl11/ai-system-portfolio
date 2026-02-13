# Escalation Flow: Real-World Example

This walkthrough shows how a task flows through the agent crew, demonstrating the escalation and de-escalation protocol in practice.

## Scenario: "Generate a report on technology decisions"

Rachel asks Nova to get a summary of recent architecture decisions and their impact.

### Step 1: Nova Creates the Issue

Nova translates the request into a structured Linear issue:

```
Title: Generate ADR impact report for Q1 2026
Label: dotti
Priority: Normal
Description:
  Query the knowledge graph for all ADRs created in 2026.
  For each ADR, include: title, status, technologies selected/rejected,
  and downstream impacts. Format as a markdown table.
```

Nova assigns the `dotti` label because this looks like a data query and report formatting task — Dotti's strength.

### Step 2: Dotti Picks Up the Issue

The `dotti` command runs, finds the labelled issue, and Dotti begins work.

```
[Dotti] Processing: Generate ADR impact report for Q1 2026
[Dotti] Querying Neo4j for Decision nodes...
[Dotti] Found 12 ADR nodes
[Dotti] Formatting markdown table...
```

Dotti successfully queries Neo4j and formats the basic table. But the "downstream impacts" part requires multi-hop graph traversal that Dotti (Haiku) struggles with — the relationships between decisions, technologies, and services form a complex subgraph.

### Step 3: Dotti Escalates to Finn

Dotti recognises it can't complete the full task and escalates:

```
[ESCALATION: dotti -> finn]

Completed: Basic ADR query and table formatting (12 ADRs found).

Blocked on: "Downstream impacts" requires multi-hop graph traversal:
  Decision -> SELECTS -> Technology -> USED_BY -> Service
  Decision -> REFERENCES -> Decision -> SELECTS -> Technology

Attempted: Single-hop queries work, but composing the multi-hop
results into a coherent impact summary exceeds my capability.

Context: Query results attached below. The graph traversal
pattern needed is described in knowledge-graph.md.
```

Dotti removes its label and adds `finn`.

### Step 4: Finn Resolves the Complex Query

Finn (Sonnet) picks up the issue, reads Dotti's context, and writes the multi-hop Cypher query:

```cypher
MATCH (d:Decision)-[:SELECTS]->(t:Technology)
OPTIONAL MATCH (t)-[:USED_BY]->(s:Service)
OPTIONAL MATCH (d)-[:REFERENCES]->(prior:Decision)
WITH d, t, collect(DISTINCT s.name) AS services,
     collect(DISTINCT prior.title) AS priorDecisions
RETURN d.title AS Decision, d.status AS Status,
       t.name AS Technology, services AS ImpactedServices,
       priorDecisions AS BuildsOn
ORDER BY d.date DESC
```

Finn generates the full report with impact analysis and marks the issue as complete.

### Step 5: De-escalation for Cleanup

If there's follow-up formatting or distribution work, Finn de-escalates back to Dotti:

```
[DE-ESCALATION: finn -> dotti]

Completed: Full impact report with multi-hop graph analysis.
Report content is in the issue comments above.

Remaining: Format the report for email distribution and
update the Linear project status.
```

## What This Demonstrates

1. **Right-sized assignment** — Nova correctly identifies this as a data task (Dotti), not a coding task (Finn)
2. **Graceful escalation** — Dotti doesn't fail silently; it documents what it completed and what it couldn't
3. **Context preservation** — the escalation comment gives Finn everything needed to continue without re-reading the original request
4. **De-escalation** — expensive model time (Finn/Sonnet) is used only for the hard part; routine follow-up flows back down
5. **Audit trail** — Linear's comment history shows exactly who did what, when, and why

## Escalation Triggers

| Trigger | From | To | Example |
|---------|------|-----|---------|
| Complex reasoning | Dotti | Finn | Multi-step logic, code generation |
| Code implementation | Dotti | Finn | Writing functions, fixing bugs |
| Architecture decisions | Finn | Sage | System design, tradeoff analysis |
| Complex debugging | Finn | Sage | Multi-file bugs, race conditions |
| Needs credentials | Any | Rachel | API keys, payment decisions |
| Business decisions | Any | Rachel | Scope changes, priority calls |

## Anti-Patterns

What the system is designed to prevent:

- **Speculative escalation** — "This looks hard, sending to Sage." Agents must *try* first.
- **Silent failure** — returning incomplete work without flagging the gap.
- **Context loss** — escalating without explaining what was attempted.
- **Upward-only flow** — everything ending up at Sage because agents don't de-escalate.
