# ADR 001: Modular Monolith

**Status:** Accepted (specification-mandated) · **Date:** 2026-10-05 · **Related:** [System Architecture](../02-system-architecture.md), [Backend Architecture](../04-backend-architecture.md)

# Context

StudentOS AI spans many domains (tasks, planning, exams, projects, focus, analytics, integrations, AI) developed by a small team that needs fast iteration, transactional consistency between related data, and low operational overhead.

# Problem

Choose a deployment and code architecture that keeps domains cleanly separated without the operational cost of distributed systems.

# Options Considered

1. **Microservices** per domain.
2. **Unstructured monolith** (single codebase, no enforced boundaries).
3. **Modular monolith** with enforced module boundaries, one database, one container image running API and worker processes.

# Decision

Modular monolith. Seventeen modules plus a `platform` kernel inside one repository (monorepo with `backend/`, `frontend/`, `infra/`, `docs/`). Modules communicate via `public.py` interfaces and domain events; import-linter enforces rules in CI.

# Reasoning

Strong consistency across planning, tasks and sessions is needed (single transactions). Team size makes distributed tracing, service discovery, versioned inter-service APIs and multiple deployables a net cost. Enforced boundaries preserve the option to extract services later.

# Trade-offs

Single failure domain and shared database scaling; discipline required to prevent boundary erosion; a heavy module (AI) can affect others unless isolated by process (worker) and queues.

# Consequences

One deployable image, two process types (API, worker). Cross-module access only through public interfaces. Boundary violations fail CI. Database is shared but tables are owned by exactly one module.

# Future Migration Path

Extract the most independent, resource-heavy modules first (`ai` processing, `google` sync workers, analytics aggregation) behind their existing public interfaces, replacing in-process calls with queue messages or HTTP. Because modules already communicate via interfaces and events and jobs are asynchronous, extraction is a deployment change. Each extraction requires its own ADR.
