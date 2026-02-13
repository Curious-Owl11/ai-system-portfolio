# Key Architecture Decisions

These are curated summaries of the most significant architecture decisions in the system. Each one involved evaluating alternatives, weighing tradeoffs, and committing to an approach. The full reasoning is preserved in internal Architecture Decision Records (ADRs).

## Multi-Agent Architecture

**Decision:** Use a tiered agent crew (Haiku / Sonnet / Opus) with label-based escalation through Linear, rather than a single model or an orchestration framework.

**Alternatives considered:**
- Single powerful model for all tasks (simpler, but expensive and wasteful)
- Orchestration framework like CrewAI or AutoGen (powerful, but adds abstraction over a simple problem)
- Custom message queue system (flexible, but over-engineered for the workload)

**Why this approach:**
- Label-based assignment through Linear means zero new infrastructure — the project management tool *is* the orchestration layer
- Tiered models match capability to task complexity, cutting costs to ~25% of the single-model approach
- Escalation comments create an audit trail — every handoff is documented with context
- De-escalation keeps expensive models focused on hard problems, not cleanup work

**Tradeoff accepted:** The system is simpler than a framework-based approach, which means it can't do things like parallel agent execution or dynamic team composition. For a solo practice, this simplicity is a feature.

## Knowledge Graph: Neo4j

**Decision:** Use Neo4j as a unified graph + vector database rather than separate graph and vector stores.

**Alternatives considered:**
- Neo4j (graph) + pgvector/Neon (vectors) — best-of-breed but requires cross-DB queries
- Memgraph — faster for real-time streaming, but the workload is batch-oriented
- Pinecone/Weaviate — purpose-built vector DBs, but adds another service with no graph relationships

**Why this approach:**
- Hybrid queries combine semantic search with graph traversal in a single Cypher statement
- "Find similar documents AND their authors AND related topics" is one query, not three
- Neo4j 5.11+ has native HNSW vector indexes — no extensions or plugins needed
- Existing Docker configuration was already set up (low migration cost)

**Tradeoff accepted:** Neo4j's vector search is newer and less battle-tested than dedicated vector databases. For the current scale (thousands of documents, not millions), this is acceptable.

## Secrets Management: Infisical

**Decision:** Centralise all secrets in Infisical (EU instance) and inject them at deploy time, rather than managing secrets per-service.

**Alternatives considered:**
- Per-service secret management (Modal secrets, Docker `.env` files) — scattered, hard to audit
- AWS Secrets Manager — vendor lock-in, more complex setup
- HashiCorp Vault — powerful but operationally heavy for a solo practice
- Just `.env` files — simple but no audit trail, rotation is manual, easy to leak

**Why this approach:**
- Single source of truth for 27+ secrets across all services
- Machine identities for automated access (no personal tokens in scripts)
- Audit log shows who accessed what and when
- Environment separation (dev/staging/prod) is built in
- One-place rotation — change a secret in Infisical, redeploy affected services

**Tradeoff accepted:** Secrets are baked at deploy time, not refreshed at runtime. For batch ETL workloads, this is fine. Long-running services would need runtime refresh.

## ETL Platform: Modal

**Decision:** Use Modal for serverless Python ETL pipelines rather than AWS Lambda, self-hosted workers, or local execution.

**Alternatives considered:**
- AWS Lambda — mature but requires CloudFormation/SAM packaging
- Self-hosted workers (Celery, etc.) — full control but always-on costs
- Local execution — no cost but can't handle GPU workloads or scale
- Temporal/Prefect — workflow orchestration platforms (more than needed)

**Why this approach:**
- `modal deploy main.py` — deployment is one command, not a CI/CD pipeline
- Pay-per-second pricing with zero idle cost (pipeline runs ~weekly)
- Native GPU access for embedding generation with larger models
- First-class Python developer experience (decorators, not YAML)

**Tradeoff accepted:** Cold start latency (~2s) and reliance on a newer platform (less community support than AWS). Acceptable for batch workloads.

## Repository Structure

**Decision:** Separate repositories by concern (docs, infrastructure, scripts, sites) rather than a monorepo.

**Alternatives considered:**
- Monorepo — everything in one place, simpler CI/CD
- More granular repos — one per service or component

**Why this approach:**
- Clear separation: documentation doesn't need the same CI/CD as infrastructure
- Independent version history — ADR changes don't pollute infrastructure commit history
- MCP access works per-repo — agents can be scoped to only the repos they need
- CI/CD per-repo: docs repo syncs to Linear, site repo deploys to Cloudflare

**Tradeoff accepted:** Cross-repo dependencies require manual coordination. For a solo practice, this is manageable.

## Infrastructure: Docker Compose with Profiles

**Decision:** Use Docker Compose with the `profiles` feature to split services into core (always running) and optional (on-demand), rather than running everything or using Kubernetes.

**Alternatives considered:**
- Always-on full stack — resource-hungry, most services idle most of the time
- Kubernetes — proper orchestration but massive overhead for a single machine
- Individual `docker run` commands — no dependency management, no shared networks
- Podman — Docker alternative but less ecosystem support

**Why this approach:**
- `docker compose up -d` starts only essential services (~200MB RAM)
- `docker compose --profile optional up -d` starts everything when needed
- Individual optional services can be started independently
- Shared networks (proxy, internal) handle inter-service communication automatically
- Named volumes ensure data persists across container restarts

**Tradeoff accepted:** Docker Compose is single-machine only. If the system needed multi-machine deployment, Kubernetes or Docker Swarm would be necessary. For a solo development environment, single-machine is the right scope.

## Cloud Strategy: Local-First, Cloud-Augmented

**Decision:** Run everything possible locally, using cloud services only for capabilities that don't work on a single machine.

| Local | Cloud | Why Cloud |
|-------|-------|-----------|
| Neo4j, PostgreSQL, Ollama | — | These work fine locally |
| — | Modal | GPU compute for embeddings |
| — | Cloudflare R2 | Durable object storage |
| — | Infisical | Secrets with audit trails |
| — | Linear | Collaborative project management |
| — | Neon | Cloud PostgreSQL for ETL targets |
| — | MotherDuck | Analytics warehouse |

**Why this approach:**
- Control — no vendor can sunset the development environment
- Speed — local queries are sub-millisecond
- Cost — most services are free to self-host
- Privacy — sensitive data stays on the machine

**Tradeoff accepted:** More operational responsibility. When something breaks, there's no vendor support to call. The upside is deep understanding of every component.
