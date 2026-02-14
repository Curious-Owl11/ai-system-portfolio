# Case Study: Relationship-Aware RAG with Neo4j

## Problem

Standard RAG (Retrieval-Augmented Generation) finds documents that are semantically similar to a query. This works well for simple lookups — "find documents about X." But it completely misses relationships between entities.

In a consulting practice, the interesting questions are relational:
- "Find research related to this topic AND show who authored it AND what decisions it informed"
- "What technologies did we evaluate alongside the one we chose?"
- "Which insights from domain X are connected to decisions in domain Y?"

A vector database can find similar documents. A graph database can traverse relationships. I needed both — in the same query.

## Approach

Instead of running a separate vector database (Pinecone, Weaviate) alongside a graph database, I used Neo4j 5.11+ which supports native HNSW vector indexes alongside traditional graph operations.

### The Hybrid Architecture

```
Documents → Parse (Docling) → Chunk → Embed (OpenAI) → Neo4j
                                                         ↓
                                              Graph Layer: nodes + relationships
                                              Vector Layer: HNSW index on embeddings
                                                         ↓
                                              Hybrid Query: vector similarity +
                                                           graph traversal in
                                                           one Cypher statement
```

Every document chunk is stored as a node with:
- **Text content** (the chunk itself)
- **Vector embedding** (1536-dimension OpenAI embedding)
- **Graph relationships** (authored by, about topic, part of document, references decision)

### Hybrid Query Pattern

The signature capability — semantic search + graph traversal in a single query:

```cypher
// Find chunks similar to the query vector
CALL db.index.vector.queryNodes('chunk_embeddings', 10, $queryVector)
YIELD node AS chunk, score

// Traverse the graph for context
MATCH (doc:Document)-[:HAS_CHUNK]->(chunk)
MATCH (doc)-[:AUTHORED_BY]->(author:Person)
OPTIONAL MATCH (doc)-[:ABOUT]->(topic:Topic)

// Return enriched results
RETURN chunk.text, score, doc.title, author.name,
       collect(DISTINCT topic.name) AS topics
ORDER BY score DESC
```

One query. One database. No cross-system joins.

### Embedding Pipeline

The embedding pipeline runs on Modal (serverless Python) to keep GPU costs pay-per-use:

1. **Extract**: Read documents from Cloudflare R2 storage
2. **Parse**: Docling handles PDF, DOCX, and other formats
3. **Chunk**: Split into semantically meaningful segments
4. **Embed**: Generate vectors via OpenAI API
5. **Load**: Write to Neo4j with graph relationships and vector embeddings

## Implementation Details

**Neo4j vector index setup:**
```cypher
CREATE VECTOR INDEX chunk_embeddings IF NOT EXISTS
FOR (c:Chunk) ON (c.embedding)
OPTIONS {indexConfig: {
  `vector.dimensions`: 1536,
  `vector.similarity_function`: 'cosine'
}}
```

**Three retrieval patterns emerged:**

| Pattern | Start With | Then | Use Case |
|---------|-----------|------|----------|
| **Semantic-first** | Vector similarity | Graph traversal for context | "Find related content and show connections" |
| **Graph-first** | Relationship traversal | Vector filtering | "Among this author's work, find similar to X" |
| **Multi-hop exploration** | Graph traversal across types | — | "What's connected to decisions that reference Y?" |

**Agent integration via MCP.** The agents don't write raw Cypher — they use the Neo4j MCP server which exposes semantic search and graph queries as structured tools. This means Dotti (Haiku) can query the knowledge graph without understanding Cypher syntax.

## Results

**Single-database simplicity.** No data sync between a vector DB and a graph DB. No impedance mismatch. No "eventual consistency" between two systems. One source of truth.

**Richer retrieval.** A standard RAG query returns "here are similar documents." A hybrid query returns "here are similar documents, written by these people, about these topics, informing these decisions." The context makes the retrieved information far more useful.

**Queryable knowledge accumulation.** As more documents are ingested, the graph grows richer. New documents automatically connect to existing entities (authors, topics, decisions). The knowledge base becomes more valuable over time without manual curation.

**Cost-effective.** Neo4j Community Edition is free. The embedding pipeline runs on Modal (pay-per-second, zero idle cost). The only ongoing cost is OpenAI embeddings for new documents.

## Lessons Learned

**Graph databases change how you think about retrieval.** Once you model knowledge as nodes and relationships, you stop thinking about "finding documents" and start thinking about "traversing knowledge." The questions you can ask change fundamentally.

**Hybrid queries are the killer feature.** Neither pure vector search nor pure graph traversal is sufficient. The combination — find semantically similar content, then traverse relationships for context — produces results that neither approach achieves alone.

**Schema design matters more than query optimisation.** The choice of node labels, relationship types, and property structures determines what questions you can ask. Getting the schema right is more important than tuning individual queries.

**Vector search in a graph DB is "good enough."** Dedicated vector databases (Pinecone, Weaviate) have more features — metadata filtering, namespace isolation, managed scaling. But for thousands of documents (not millions), Neo4j's native vector index is perfectly adequate, and the graph integration more than compensates.

**Start with the questions you want to ask.** I designed the schema by listing the queries I'd want to run — "find related content by this author," "what did we evaluate alongside X," "what's connected to this decision." The schema fell out of the question design, not the other way around.
