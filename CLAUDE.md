# CLAUDE.md: Rules for AI Agents and Contributors

**Read this file first in every session.** It is the operating manual for Claude Code and human contributors working on StudentOS AI.

## 1. Current Status

Architecture and documentation are complete (v0.1.0-docs). Application code may only be written after the user explicitly approves implementation, and only for the phase and slice that was approved ([Development Roadmap](docs/23-development-roadmap.md)).

## 2. Project Architecture

StudentOS AI is a **modular monolith**: React SPA → FastAPI API → application modules → PostgreSQL (+ pgvector), with a Postgres-backed background worker, S3-compatible object storage, an AI Gateway and Google integrations.

- Overview: [System Architecture](docs/02-system-architecture.md)
- Modules and layering: [Backend Architecture](docs/04-backend-architecture.md)
- Frontend: [Frontend Architecture](docs/03-frontend-architecture.md)
- Schema: [Database Design](docs/05-database-design.md); API: [API Specification](docs/06-api-specification.md)
- Decisions: [docs/adr](docs/adr/001-modular-monolith.md)

Core intelligence rule: **deterministic engines decide, the LLM proposes and explains.** Student data → Constraint Engine → Priority Engine → Scheduling Engine → AI Explanation Layer → student ([Task Planning Engine](docs/10-task-planning-engine.md)).

## 3. Repository Structure (planned)

```text
backend/app/<module>/{api,service,domain,repository,schemas,jobs}   one folder per module
backend/app/platform/        config, db, logging, errors, security, jobs, storage
backend/alembic/             migrations
backend/tests/{unit,integration,api,authz,ai}
frontend/src/{app,features,shared}    features/<module> mirror backend modules
frontend/e2e/                Playwright
infra/                       Dockerfiles, docker-compose, deploy config
docs/                        source of truth for design
```

Modules: `identity, profile, academics, tasks, planning, exams, projects, hackathons, focus, analytics, recommendations, resources, notifications, gamification, google, ai, admin` plus the shared `platform` kernel.

## 4. Required Make Interface (Phase 1 must provide)

`make dev` (all services), `make test`, `make lint`, `make typecheck`, `make migrate`, `make openapi` (regenerate spec and TS client), `make e2e`. CI calls these same targets.

## 5. Architectural Change Protocol (most important rule)

**Claude Code must NOT silently make major architectural decisions.** Major = new infrastructure component, new language/framework/library with architectural reach, changing a module boundary, auth/token design, data model strategy, AI provider coupling, or anything contradicting an ADR.

Before any such change:

1. Explain the change.
2. Explain why it is required.
3. Explain alternatives.
4. Explain trade-offs.
5. Update the relevant documentation.
6. Create or update an ADR.
7. Implement only after the user approves.

When the spec is ambiguous: state the ambiguity, list alternatives, recommend one, label it "architectural recommendation", and record it. Never invent major requirements silently.

## 6. Development Workflow

1. Identify the roadmap phase and read every doc linked from it.
2. Restate the slice (DB, backend, API, frontend, tests, docs) and confirm scope.
3. Write migration → models/repositories → domain/service → routes/schemas → frontend → tests → docs.
4. Run `make lint typecheck test` before declaring done.
5. Update the API spec, DB design and CHANGELOG in the same change.

## 7. Coding Principles

- Do not over-engineer; do not add microservices; no duplicated logic.
- Layering inside each module: **router → service → domain/repository**. Routers do HTTP only. Services hold use cases and transactions. Domain holds pure logic (no I/O). Repositories hold SQLAlchemy.
- A module never imports another module's models, repositories or internals. It uses the other module's `public.py` interface or domain events. Enforced by import-linter.
- Python: full type hints, mypy strict on `app/`, Ruff, Pydantic v2 schemas at every boundary. TypeScript: `strict`, no `any`, no unexplained `@ts-ignore`.
- Server state lives in TanStack Query; Zustand only for client-only UI state (auth token in memory, UI preferences, local timer rendering).
- Structured logging only (no `print`), no PII in logs.
- Time: store UTC `timestamptz`; the user's IANA timezone is applied only at the edges (engine inputs, display).
- Avoid unnecessary dependencies; justify each new one in the PR.

## 8. Security Rules

- Never hardcode or commit secrets; never commit `.env`.
- Every user-owned query is scoped by `user_id` in the repository layer; foreign resources return **404**, not 403 ([Authorization](docs/08-authorization.md)).
- `user_id` is never accepted from request bodies or query strings.
- Admins must not see private student content by default; use only aggregates and metadata, plus consented break-glass access.
- Validate all input (Pydantic), all external API responses, all uploaded files, all AI output.
- Passwords: Argon2id only. Tokens: see [Authentication](docs/07-authentication.md). Google tokens are encrypted at rest.
- Treat Classroom text, uploaded documents and web content as untrusted data in prompts ([AI Architecture](docs/09-ai-architecture.md)).
- Details: [Security](docs/17-security.md), [Privacy](docs/18-privacy.md).

## 9. Database Rules

- Every schema change is an Alembic migration; never edit applied migrations; migrations are backward compatible with the previous release (expand/contract).
- Conventions in [Database Design](docs/05-database-design.md): UUIDv7 ids, `timestamptz`, `text` + CHECK instead of PG enums, `user_id` on every user-owned table, composite ownership FKs, soft delete via `deleted_at` where documented, indexes for every FK and common filter.
- No raw string-built SQL. No large blobs in PostgreSQL.
- Never use `SELECT *` in repositories; paginate every list query.

## 10. API Rules

- Base path `/api/v1/`; breaking changes need a new version.
- Consistent error body, cursor pagination, `Idempotency-Key` on documented POSTs, `202 Accepted` for async work ([API Specification](docs/06-api-specification.md)).
- Response schemas are explicit Pydantic models; never return ORM objects directly.
- The OpenAPI spec is generated from code; the frontend client is generated from the spec; CI fails on drift.

## 11. AI Rules

- All LLM/embedding calls go through the AI Gateway; no feature imports a provider SDK.
- **Never trust or execute raw AI output.** Pipeline: LLM → schema validation → business validation → accept / retry / reject.
- The LLM never decides schedule times, priorities or deletions. AI results are proposals the user accepts.
- Prompts are versioned files; every request is logged (metadata always; payload encrypted with short retention); per-user quotas and a global budget breaker apply.
- Tests use the fake provider; no real LLM calls in CI unit/integration runs.

## 12. Testing Requirements

New logic needs tests. Domain engines need unit and property tests. Every new resource endpoint needs an authorization test proving user B cannot read or modify user A's data. AI features need schema-validation tests with malformed fixtures. See [Testing Strategy](docs/19-testing-strategy.md).

## 13. Documentation Requirements

Update in the same PR as the code: API spec, DB design, relevant module doc, ADR (if decision-level), CHANGELOG. Relative links must resolve. Mermaid diagrams must match the real architecture.

## 14. Git Workflow

Trunk-based with short-lived branches: `feat/…`, `fix/…`, `docs/…`, `chore/…`. Conventional Commits. PRs require green CI and review. Squash merge to `main`. See [CONTRIBUTING.md](CONTRIBUTING.md).

## 15. Definition of Done

A feature is done only when ALL hold: database implemented + migration created + backend implemented + API documented + frontend implemented + validation implemented + authorization implemented + tests written + error handling implemented + documentation updated.

## 16. Ask Before

Adding infrastructure; changing auth or token design; adding a dependency with a runtime footprint; touching Google scopes; changing retention or privacy behaviour; sending new data categories to an AI provider; deleting or rewriting migrations; starting a phase that was not approved.
