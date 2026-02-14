# Case Study: Operational Memory for AI Agents

## Problem

Stateless agents repeat mistakes. Every invocation starts fresh — no memory of what was tried before, what worked, what failed. For a crew of agents processing Linear issues daily, this means:

- Finn might attempt the same failing approach on a recurring bug pattern
- Dotti can't learn that certain query formats produce better results
- No agent knows "we tried this last week and it didn't work"

LLM context windows provide conversation memory within a session, but nothing persists between sessions. The agents needed operational memory — a fast-lookup store of past tasks, approaches, and outcomes that persists across invocations.

## Approach

I built a lightweight memory system for Finn (the coding agent) using FAISS for fast vector similarity search and SQLite for durable task history.

### Architecture

```
New Task Arrives
      ↓
Generate embedding (sentence-transformers, 384 dims)
      ↓
FAISS similarity search → "Have I seen something like this before?"
      ↓
If similar tasks found:
  → Retrieve approaches, outcomes, context
  → Inform current attempt ("last time this pattern appeared, X worked")
      ↓
After task completion:
  → Store task, approach, outcome as new memory
  → FAISS index updated, SQLite row inserted
```

### Why FAISS + SQLite (Not a Database)?

| Approach | Pros | Cons |
|----------|------|------|
| **FAISS + SQLite** | Sub-millisecond search, zero dependencies, local-only | No built-in replication |
| **Neo4j vectors** | Unified with knowledge graph | Network round-trip, heavier for simple lookups |
| **pgvector** | Familiar SQL | Overkill for single-agent memory |
| **In-memory dict** | Simplest | Lost on restart |

The memory system needs to be fast (checked on every task), lightweight (runs alongside the agent), and persistent (survives restarts). FAISS handles similarity search at microsecond speed. SQLite handles persistence with zero configuration.

## Implementation Details

**Embedding model:** `all-MiniLM-L6-v2` — a 384-dimension sentence-transformer that runs locally. No API calls, no cost, no latency. Good enough for task-level similarity (not document-level semantic search, which uses OpenAI embeddings in Neo4j).

**Core operations:**

```python
# Store a completed task
memory.store_task(
    description="Fix pagination bug in API endpoint",
    approach="Off-by-one error in OFFSET calculation",
    outcome="success",
    metadata={"issue": "CORE-456", "files": ["api/routes.py"]}
)

# Find similar past tasks
similar = memory.find_similar_tasks(
    "Pagination returns duplicate results",
    top_k=3
)
# Returns: previous pagination fixes with approaches and outcomes

# Recent task history
recent = memory.get_recent_tasks(limit=10)

# Usage statistics
stats = memory.get_stats()
```

**Fallback behaviour:** If FAISS isn't available (missing dependency), the system falls back to SQLite `LIKE` search on task descriptions. Degraded but functional — the agent still has memory, just without semantic similarity.

**Storage footprint:** SQLite database + FAISS index together are typically under 10MB for thousands of tasks. Negligible compared to the LLM context window.

## Results

**Pattern recognition across sessions.** When a similar issue appears, the agent can reference what worked before. "The last three times a pagination bug was reported, the root cause was OFFSET calculation" is more useful than starting from scratch.

**Reduced redundant exploration.** Without memory, agents sometimes try approaches that previously failed. With memory, the agent can check "has this approach been tried on similar tasks?" before committing to it.

**Task-level learning without fine-tuning.** The agent doesn't need model fine-tuning to improve — it just needs to check its memory. This is retrieval-augmented behaviour, not model modification.

**Measurable through crew-metrics.** The existing `crew-metrics` command tracks resolution times and escalation rates, so the impact of agent memory on performance is observable over time.

## Lessons Learned

**The right embedding model depends on the use case.** For task-level similarity (short descriptions, specific patterns), a small local model (`all-MiniLM-L6-v2`, 384 dims) is better than a large cloud model. It's faster, free, and the similarity quality is more than adequate for matching task descriptions.

**Graceful degradation is worth the effort.** The FAISS fallback to SQLite LIKE search took minimal code but means the system works even in environments where FAISS can't be installed. Build the optimal path, but always have a functional fallback.

**Agent memory is a different problem from knowledge retrieval.** The knowledge graph (Neo4j) stores organisational knowledge — documents, decisions, relationships. Agent memory stores operational experience — what was tried, what worked, what failed. These serve different purposes and benefit from different storage strategies.

**Lightweight wins.** The entire memory system is two files — `memory.py` and `models.py`. No framework, no server, no configuration. It runs alongside the agent as a library, not as a service. For a single-agent memory system, this level of simplicity is appropriate.
