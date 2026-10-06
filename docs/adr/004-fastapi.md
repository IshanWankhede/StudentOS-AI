# ADR 004: FastAPI

**Status:** Accepted (specification-mandated) · **Date:** 2026-10-05 · **Related:** [Backend Architecture](../04-backend-architecture.md), [API Specification](../06-api-specification.md)

# Context

The backend needs a typed, async-capable Python web framework with automatic OpenAPI generation, strong validation, and good fit for the AI ecosystem.

# Problem

Pick the web framework and conventions that support a typed contract between backend and frontend.

# Options Considered

1. **FastAPI** with Pydantic v2.
2. Django + Django REST Framework.
3. Flask.
4. A Node.js/TypeScript backend (NestJS).

# Decision

FastAPI with Pydantic v2 schemas, async SQLAlchemy 2.x, dependency injection via `Depends`, an OpenAPI document generated in CI from which the TypeScript client is generated. Routers contain no business logic ([Backend Architecture](../04-backend-architecture.md#3-layering-rules)).

# Reasoning

Automatic schemas give a verified API contract and typed frontend client. Async fits I/O-heavy work (database, LLM, Google). Python is the natural language for AI and scheduling logic, and Pydantic doubles as the AI structured-output validator.

# Trade-offs

Less batteries-included than Django (no built-in admin or auth; we implement auth deliberately); async SQLAlchemy has pitfalls (lazy loading, session scope) that need documented rules; two languages across the stack.

# Consequences

Requires explicit layering discipline, typed settings, import-linter, and mypy strict mode. The admin console is custom and deliberately limited. Procrastinate workers reuse the same services.

# Future Migration Path

If the framework must change, routers are thin and schemas are Pydantic models, so only the HTTP layer would be replaced. If sync-only needs arise (rare), SQLAlchemy services can run in thread pools. API versioning (`/api/v1`) allows incremental replacement.
