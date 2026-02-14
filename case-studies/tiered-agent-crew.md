# Case Study: Tiered Agent Crew with Autonomous Escalation

## Problem

Running a solo consulting practice means wearing every hat — admin, developer, analyst, architect. AI agents can handle much of this work, but a single-model approach creates a dilemma: use a powerful (expensive, slow) model for everything, or use a cheap (fast, limited) model and accept failures on complex tasks.

Neither extreme works. Most daily tasks are simple — run a query, format a report, update a status. But some require genuine reasoning — debug a multi-service issue, design a system, evaluate technology tradeoffs. A flat approach either overpays for simple work or under-delivers on hard problems.

## Approach

I built a three-tier agent crew where each agent runs a different Claude model matched to its role:

| Agent | Model | Cost Tier | Handles |
|-------|-------|-----------|---------|
| Dotti | Haiku | Lowest | Data queries, reports, admin |
| Finn | Sonnet | Mid | Code, implementation, bug fixes |
| Sage | Opus | Highest | Architecture, complex debugging |

The key mechanism is **autonomous escalation through Linear labels**. Each agent watches for issues tagged with its name. When an agent encounters a task beyond its capability, it:

1. Documents what it completed and where it got stuck
2. Removes its own label from the issue
3. Adds the next agent's label
4. Posts a structured handoff comment

```
[ESCALATION: dotti -> finn]

Completed: Basic ADR query and table formatting (12 ADRs found).
Blocked on: Multi-hop graph traversal exceeds reasoning capability.
Attempted: Single-hop queries work, composing results does not.
Context: Query results attached. See knowledge-graph.md for traversal patterns.
```

After the higher-tier agent resolves the complex part, it **de-escalates** routine follow-up work back down — keeping expensive model time focused on hard problems.

## Implementation Details

**Orchestration is just labels.** No message queue, no custom API, no framework. Linear (the project management tool) is the coordination layer. This was a deliberate decision — the simplest mechanism that works is the best one.

**Each agent runs via PowerShell:**
```
crew              # Runs all three in sequence: dotti → finn → sage
dotti             # Processes only dotti-labelled issues
sage              # Processes only sage-labelled issues
crew-metrics      # Escalation statistics and patterns
```

**Metrics tracking** logs every run — duration, issue count, escalation events, resolution outcomes. This data feeds back into assignment heuristics: if Dotti consistently escalates a certain task type, the assignment guide gets updated.

**MCP integration** gives every agent access to the knowledge graph (Neo4j), project management (Linear), business data (Fibery), and research (Zotero) — through standardised tool interfaces, not custom API code.

## Results

- **~75% cost reduction** compared to running everything on Opus
- **80% of tasks** resolve at the Haiku tier without escalation
- **Structured handoffs** mean zero context loss during escalation — the receiving agent has everything it needs
- **De-escalation** prevents the common anti-pattern of everything drifting upward to the most expensive model
- **Audit trail** — every handoff is documented in Linear with timestamps and reasoning

## Lessons Learned

**Model selection is an engineering decision, not a default.** The instinct is to use the best model for everything "just in case." But matching model capability to task complexity is the same principle as right-sizing compute instances — you don't run a GPU cluster to serve a static website.

**Coordination mechanisms should be boring.** I evaluated CrewAI, AutoGen, and custom orchestration before realising that label swapping on an existing project management tool was sufficient. The simplest mechanism that solves the problem is the right one.

**Structured handoffs are non-negotiable.** Early experiments with unstructured escalation ("this is too hard, sending to Finn") produced poor results. The receiving agent wasted time re-reading the original issue and repeating failed approaches. The structured format — completed/blocked/attempted/context — eliminated this.

**De-escalation matters as much as escalation.** Without explicit de-escalation, every task eventually ends up at the most capable (and expensive) agent, even for trivial follow-up work. The protocol explicitly routes cleanup work back down.
