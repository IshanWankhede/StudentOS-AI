# ADR 002: PostgreSQL

**Status:** Accepted (specification-mandated) · **Date:** 2026-10-05 · **Related:** [Database Design](../05-database-design.md)

# Context

The system is relational (users own subjects, tasks, subtasks, sessions, plans) and needs transactions, constraints, vector search, a job queue, and strong tooling.

# Problem

Select the primary data store, and whether to use additional stores for queues, search or vectors.

# Options Considered

1. **PostgreSQL** with extensions (pgvector, citext, btree_gist).
2. MySQL/MariaDB.
3. Document store (MongoDB) plus separate vector database.
4. PostgreSQL plus Redis plus a dedicated vector store.

# Decision

PostgreSQL 16+ as the single system of record, also used for the job queue (see [ADR 009](009-background-jobs.md)) and vector similarity ([ADR 003](003-pgvector.md)). SQLAlchemy 2.x with Alembic migrations. `text` + CHECK constraints instead of PG enum types; UUIDv7 primary keys; composite ownership foreign keys.

# Reasoning

Relational integrity (composite FKs, exclusion constraints, partial unique indexes) enforces ownership and scheduling invariants in the database itself. One datastore minimises operations. Mature backup/PITR tooling is widely available on managed services.

# Trade-offs

Using Postgres as a queue and vector store limits peak throughput compared with specialised systems; managed-provider extension support must be verified; single primary scales vertically first.

# Consequences

All state, including jobs, is backed up and restored together. Migrations are the only way to change schema. Requires pgvector and `btree_gist` support on the chosen host.

# Future Migration Path

Add PgBouncer, then a read replica for analytics. Move the queue to Redis/SQS or vectors to a dedicated engine behind the existing job and similarity interfaces only when metrics justify it. Partition large append-only tables (`audit_logs`, `ai_requests`) by time if needed.
