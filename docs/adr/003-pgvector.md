# ADR 003: pgvector

**Status:** Accepted (specification-mandated) · **Date:** 2026-10-05 · **Related:** [AI Architecture](../09-ai-architecture.md), [Recommendation Engine](../12-recommendation-engine.md)

# Context

Semantic similarity is needed for resource search, matching tasks to a student's saved resources, and later news and video features. Data is strictly per user and small per user.

# Problem

Provide vector search without adding a separate datastore, and decide embedding dimension and indexing strategy.

# Options Considered

1. **pgvector** inside PostgreSQL.
2. Dedicated vector database (Qdrant, Pinecone, Weaviate).
3. Keyword search only.

# Decision

pgvector with `vector(1024)` columns on `resources` and `tasks` (nullable, introduced in Phase 15) and later on news/video tables. Embeddings are produced through the Embedding Service in the AI Gateway. Queries always filter by `user_id` first and use an exact distance scan; HNSW indexes are added only when measured latency requires it. The extension is installed in Phase 1.

# Reasoning

Per-user datasets (hundreds to low thousands of rows) make exact scans fast and avoid index memory costs and filtered-search recall issues. Transactional consistency with source rows is automatic, and ownership rules apply unchanged.

# Trade-offs

Dimension is fixed per column: switching to a model with a different dimension requires a re-embedding migration. Global content (news) at large scale may eventually need ANN indexes or a dedicated engine. Managed-host support for the chosen pgvector version must be verified.

# Consequences

Embedding model name is stored per row (`embedding_model`) so mixed or stale embeddings are detectable. The chosen embedding model must support 1024 dimensions (native or truncation). Embedding jobs run in the worker.

# Future Migration Path

Add HNSW (or IVFFlat) indexes for global tables first. If cross-user or very large corpora appear, move similarity behind the same `similar(...)` interface to a dedicated vector database and backfill from PostgreSQL. A model change follows: add new column, dual-write, backfill, switch reads, drop the old column.
