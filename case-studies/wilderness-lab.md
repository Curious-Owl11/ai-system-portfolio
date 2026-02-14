# Case Study: Wilderness Lab — Multi-Agent Epistemological Experiments

## Problem

Single-perspective AI outputs have a blind spot: they reflect one reasoning style. Ask a question and you get one coherent answer — but you don't see what an empiricist would challenge, what a rationalist would derive differently, or what a pragmatist would prioritise instead.

For consulting work — especially in strategy and systems thinking — this single-perspective limitation is a real problem. Good decisions require examining ideas from multiple angles. I wanted to explore whether structuring multi-agent conversations around distinct *epistemological stances* could surface insights that a single agent misses.

## Approach

I built Wilderness Lab — a framework for running structured multi-agent conversations where each agent embodies a different philosophical approach to knowledge.

### The Four-Layer Composition System

The framework separates concerns into composable layers:

```
Layer 1: Domains     → What knowledge context (marketing, AI ethics, physics)
Layer 2: Personas    → What thinking style (empiricist, rationalist, pragmatist)
Layer 3: Protocols   → How they interact (round-robin, debate, consensus)
Layer 4: Runtimes    → Where it executes (Anthropic API, CAMEL-AI framework)
```

Any experiment is a composition: pick a domain, assign personas, choose a protocol, select a runtime. The same personas work across domains — they're portable thinking styles, not domain-specific characters.

### Persona Design

Each persona is defined by an epistemic stance, a primary question, characteristic behaviours, and challenge phrases:

**Empiricist:**
- Stance: "Knowledge grounded in observation and sensory experience"
- Primary question: "What evidence supports this?"
- Behaviours: Demands observable evidence, questions assumptions, proposes experiments

**Rationalist:**
- Stance: "Knowledge grounded in reason and logical deduction"
- Primary question: "What follows logically from our premises?"
- Behaviours: Examines assumptions, builds from first principles, uses thought experiments

**Pragmatist:**
- Stance: "Knowledge evaluated by practical consequences"
- Primary question: "What difference does it make?"
- Behaviours: Tests ideas against real-world application, focuses on outcomes

**Constructivist:**
- Stance: "Knowledge is socially constructed"
- Primary question: "Whose perspective is this?"
- Behaviours: Considers context, surfaces hidden assumptions about who benefits

### The Council Pattern

The most interesting configuration runs all four personas in round-robin on the same question. Each agent responds from its epistemic stance, directly engaging with what previous agents said:

```
Round 1: Empiricist makes initial evidence-based claim
         Rationalist challenges logical foundations
         Pragmatist asks "but does it work in practice?"
         Constructivist asks "whose evidence counts?"

Round 2: Each agent responds to the challenges raised
         Positions evolve, unexpected agreements emerge
```

## Implementation Details

**Built with Python + Pydantic schemas** for strict typing on all four layers. Experiments are defined in YAML and composed at runtime.

**CAMEL-AI integration** provides the multi-agent conversation runtime, handling turn-taking and message passing between agents.

**Neo4j tracking** stores every experiment run as a graph:
- `Run` nodes (experiment executions with timestamps)
- `Turn` nodes (individual agent responses)
- `Observation` nodes (patterns I notice: position changes, unexpected agreements)
- `Insight` nodes (promoted observations that become reusable knowledge)

This means I can query: "In which domains does the Empiricist/Rationalist pairing produce the most position changes?" — a graph query across all experiment history.

**Tested with:** AI consciousness debate ("How should we evaluate whether AI systems are conscious?") as the first full experiment — chosen because it naturally surfaces disagreement between empirical and rationalist approaches.

## Results

**Productive tension by design.** The Empiricist/Rationalist pairing consistently produces substantive disagreement that neither agent alone would surface. The Empiricist demands "what would we observe?" while the Rationalist insists on logical coherence — forcing each to sharpen their position.

**Portable personas.** The same thinking styles work across domains (marketing, ethics, physics, philosophy) without modification. An Empiricist asking "what evidence?" is useful whether the topic is ad spend or consciousness.

**Emergent synthesis.** In multi-round conversations, agents don't just repeat their positions — they evolve. The Pragmatist often finds middle ground that the Empiricist and Rationalist missed because they were focused on their epistemological clash.

**Graph-queryable experiment history.** Storing runs in Neo4j means patterns accumulate over time. After enough experiments, the graph reveals which persona combinations and which protocols produce the most diverse, highest-quality outputs.

## Lessons Learned

**Persona design is more important than prompt engineering.** The quality of multi-agent conversations depends on how well the personas are differentiated — not on how clever the prompts are. Clear epistemic stances, specific challenge phrases, and distinct primary questions produce better results than elaborate system prompts.

**Protocols shape outcomes.** The same personas with the same question produce different results under different protocols. Round-robin (everyone speaks in turn) produces more diverse perspectives. Debate (point/counterpoint) produces deeper analysis of fewer positions. The protocol is a first-class design decision.

**Observation tracking changes how you think about experiments.** By storing observations as graph nodes linked to runs, personas, and domains, patterns become queryable rather than anecdotal. "The Empiricist tends to concede on X-type questions" becomes a data point, not a feeling.

**Composition beats configuration.** Separating domains, personas, protocols, and runtimes into independent layers means experiments are easy to vary systematically. Change one layer, keep the rest fixed, observe the difference. This is basic experimental design applied to multi-agent systems.
