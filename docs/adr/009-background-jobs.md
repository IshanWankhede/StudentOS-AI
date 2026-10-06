# ADR 009: Background Jobs

**Status:** Accepted (architectural recommendation; the specification requires a background worker but names no technology) · **Date:** 2026-10-05 · **Related:** [Backend Architecture](../04-backend-architecture.md#7-background-jobs), [Notification System](../14-notification-system.md)

# Context

Asynchronous work: AI processing, notifications, Calendar/Classroom sync, file scanning, recommendation generation, analytics aggregation, news ingestion, exports, purges. Principles: prefer simple architecture and avoid unnecessary dependencies.

# Problem

Select a job system that is reliable, observable and cheap to operate at MVP scale.

# Options Considered

1. **PostgreSQL-backed queue (Procrastinate)**.
2. Celery + Redis (or RabbitMQ).
3. ARQ or Dramatiq + Redis.
4. Cloud-native queues (SQS, Cloud Tasks) plus functions.
5. A hand-written `jobs` table with `SKIP LOCKED`.

# Decision

Procrastinate on the existing PostgreSQL database, queues `ai`, `sync`, `notifications`, `maintenance`, `default`, run by the same container image as the API with a different command. Periodic tasks for scanning and maintenance. Jobs are idempotent, carry `correlation_id`, retry with backoff, and have a visible dead state. A sweeper re-enqueues rows (for example `ai_requests` stuck in `queued`) to cover failure between commit and enqueue. Rate limiting for auth and AI uses database counters; generic limits sit at the edge. **Verify Procrastinate's current feature set and compatibility before Phase 6.**

# Reasoning

No new infrastructure to run, back up or secure; the queue is as durable as the data; async-native; periodic tasks included; observable through SQL. Scale expectations (tens of thousands of jobs per day) are far within PostgreSQL's capability.

# Trade-offs

Lower peak throughput than Redis-based systems; enqueue is not in the same transaction as the application's SQLAlchemy session by default (mitigated by the sweeper and idempotency); smaller community than Celery; distributed rate limiting is weaker without Redis.

# Consequences

Job payloads are small and versioned (handlers accept N and N-1). Worker scaling is by process count. Queue depth and oldest-job age are first-class alerts ([Observability](../20-observability.md)). Redis is intentionally absent from MVP.

# Future Migration Path

Wrap job submission in a thin `enqueue(job_name, payload, queue)` interface owned by `platform`. If throughput, scheduling needs or distributed rate limiting demand it, introduce Redis (ARQ/Celery) or a cloud queue, dual-run during cutover, and move queues one at a time starting with `notifications` or `sync`.
