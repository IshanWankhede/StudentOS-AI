# StudentOS AI

> An intelligent personal operating system for college students: academics, schedules, projects, hackathons, learning, productivity and career work in one place, with AI that helps students decide **what to work on right now, why, how, and when**.

**Status:** Architecture and documentation phase complete (v0.1.0-docs). **No application code exists yet.** Implementation starts only after explicit approval of [the architecture review](docs/25-architecture-review.md).

---

## 1. Overview

StudentOS AI replaces a patchwork of Classroom, Calendar, to-do apps, notes, Pomodoro timers and hackathon trackers with one system built around a closed feedback loop:

```text
Student context → workload → goals → deadlines → available time → prioritise
→ break work into steps → roadmap → schedule study sessions → track execution
→ analyse progress → better future recommendations
```

It is **not** a to-do list with an LLM attached. Scheduling and prioritisation are deterministic engines; the LLM proposes task breakdowns, explains plans and personalises recommendations, and every AI output is validated before the application uses it ([AI Architecture](docs/09-ai-architecture.md), [Task Planning Engine](docs/10-task-planning-engine.md)).

## 2. Features

| Tier | Features |
|---|---|
| **MVP** | Email + Google login, onboarding (profile, interests, goals, availability), subjects, tasks, subtasks and dependencies, AI task breakdown, roadmap generation, timetable, Pomodoro and study sessions, basic analytics and dashboard, exams with basic exam roadmap, projects (GitHub URL), hackathon workspace, Google Calendar sync, basic Google Classroom sync, in-app notifications, AI recommendations, minimal admin |
| **Post-MVP** | Advanced recommendations, YouTube assistant, hackathon recommendations, AI news, gamification, advanced analytics, Google Drive, email notifications, advanced exam planner, adaptive planning |
| **Future** | Android digital wellbeing, ML recommender, voice assistant, teacher accounts, college dashboards, social features, AI agents, mobile app, cross-college analytics |

Details: [Product Requirements](docs/01-product-requirements.md), [Future Roadmap](docs/24-future-roadmap.md).

## 3. Architecture Overview

A **modular monolith** (no microservices): one React SPA, one FastAPI backend with strictly bounded internal modules, one background worker process, PostgreSQL (with pgvector) and S3-compatible object storage.

```text
Browser (React SPA)
      ↓  HTTPS /api/v1
FastAPI API  ──→  Application modules  ──→  PostgreSQL + pgvector
      │                  │
      │                  ├──→ AI Gateway ──→ LLM / Embedding providers
      │                  ├──→ Google APIs (OAuth, Calendar, Classroom)
      │                  └──→ Object storage
      └──→ Background worker (Postgres-backed queue)
```

See [System Architecture](docs/02-system-architecture.md) and [ADR 001](docs/adr/001-modular-monolith.md).

## 4. Technology Stack

| Layer | Technology |
|---|---|
| Frontend | React, TypeScript (strict), Vite, TanStack Query, Zustand, React Router, Tailwind CSS, React Hook Form + Zod |
| Backend | Python 3.12+, FastAPI, Pydantic v2, SQLAlchemy 2.x (async), Alembic |
| Database | PostgreSQL 16+ with pgvector |
| Background jobs | Procrastinate (PostgreSQL-backed queue), see [ADR 009](docs/adr/009-background-jobs.md) |
| Object storage | S3-compatible (S3, R2, MinIO locally) |
| AI | Provider-agnostic AI Gateway (LLM + embeddings) |
| Integrations | Google OAuth/OIDC, Google Calendar API, Google Classroom API (Drive post-MVP) |
| Quality | pytest, Vitest, Testing Library, Playwright, Ruff, mypy, ESLint |
| Delivery | Docker, GitHub Actions |

Rationale for each choice lives in the [ADRs](docs/adr/).

## 5. Repository Structure

Planned layout (created during implementation, not yet present):

```text
studentos-ai/
├── backend/        FastAPI modular monolith, Alembic migrations, tests
├── frontend/       React SPA, unit/component tests, Playwright e2e
├── infra/          Dockerfiles, docker-compose for local development, deploy config
├── .github/        GitHub Actions workflows
└── docs/           Architecture and product documentation (this repository today)
```

## 6. Documentation Index

| # | Document | Topic |
|---|---|---|
| 01 | [Product Requirements](docs/01-product-requirements.md) | Scope, requirements, workflows |
| 02 | [System Architecture](docs/02-system-architecture.md) | Overall architecture |
| 03 | [Frontend Architecture](docs/03-frontend-architecture.md) | SPA structure and state |
| 04 | [Backend Architecture](docs/04-backend-architecture.md) | Modules, layering |
| 05 | [Database Design](docs/05-database-design.md) | Schema, ER diagram |
| 06 | [API Specification](docs/06-api-specification.md) | `/api/v1` endpoints |
| 07 | [Authentication](docs/07-authentication.md) | Login, tokens, sessions |
| 08 | [Authorization](docs/08-authorization.md) | RBAC, ownership |
| 09 | [AI Architecture](docs/09-ai-architecture.md) | AI Gateway, validation |
| 10 | [Task Planning Engine](docs/10-task-planning-engine.md) | Priority, scheduling |
| 11 | [Exam Planning Engine](docs/11-exam-planning-engine.md) | Exam roadmaps |
| 12 | [Recommendation Engine](docs/12-recommendation-engine.md) | Recommendations |
| 13 | [Google Integrations](docs/13-google-integrations.md) | OAuth, Calendar, Classroom |
| 14 | [Notification System](docs/14-notification-system.md) | Notifications |
| 15 | [Gamification](docs/15-gamification.md) | XP, streaks (post-MVP) |
| 16 | [File Storage](docs/16-file-storage.md) | Uploads, object storage |
| 17 | [Security](docs/17-security.md) | Security controls |
| 18 | [Privacy](docs/18-privacy.md) | Data ownership, deletion |
| 19 | [Testing Strategy](docs/19-testing-strategy.md) | Test plan |
| 20 | [Observability](docs/20-observability.md) | Logs, metrics, alerts |
| 21 | [Deployment](docs/21-deployment.md) | Environments, backups |
| 22 | [CI/CD](docs/22-ci-cd.md) | GitHub Actions |
| 23 | [Development Roadmap](docs/23-development-roadmap.md) | 22 phases |
| 24 | [Future Roadmap](docs/24-future-roadmap.md) | Post-MVP and future |
| 25 | [Architecture Review](docs/25-architecture-review.md) | Gaps, risks, recommendations |

ADRs: [001 Modular monolith](docs/adr/001-modular-monolith.md) · [002 PostgreSQL](docs/adr/002-postgresql.md) · [003 pgvector](docs/adr/003-pgvector.md) · [004 FastAPI](docs/adr/004-fastapi.md) · [005 Frontend stack](docs/adr/005-frontend-stack.md) · [006 Authentication](docs/adr/006-authentication-strategy.md) · [007 Google OAuth](docs/adr/007-google-oauth.md) · [008 AI provider abstraction](docs/adr/008-ai-provider-abstraction.md) · [009 Background jobs](docs/adr/009-background-jobs.md) · [010 Object storage](docs/adr/010-object-storage.md)

Contributor rules for humans and AI agents: [CLAUDE.md](CLAUDE.md), [CONTRIBUTING.md](CONTRIBUTING.md), [SECURITY.md](SECURITY.md).

## 7. Development Roadmap

Twenty-two phases from project setup to production polish; MVP release = phases 1–16 and 18–22 (gamification, phase 17, is post-MVP). See [Development Roadmap](docs/23-development-roadmap.md). The recommended first vertical slice is **email authentication and session management on a fully scaffolded stack**.

## 8. Local Development Prerequisites

Required once implementation begins:

- Python 3.12+ and a dependency manager (uv recommended)
- Node.js current LTS and a package manager (pnpm recommended)
- Docker with Compose (PostgreSQL + pgvector, MinIO, MailHog-compatible SMTP sink)
- Git

## 9. Environment Setup Overview

Copy `.env.example` to `.env` and fill in values. Real secrets are never committed. Local development uses Docker Compose for PostgreSQL and MinIO, a fake LLM provider by default (no API key needed), and Google OAuth credentials only when working on Google features. See [Deployment](docs/21-deployment.md#3-environment-variables).

## 10. Testing Overview

Unit, integration, API, database, authentication, authorization and AI-schema tests on the backend; component, integration and Playwright E2E tests on the frontend. The critical E2E journey is Register → Onboarding → Create Task → AI Breakdown → Roadmap → Pomodoro → Complete → Dashboard. See [Testing Strategy](docs/19-testing-strategy.md).

## 11. Security Notes

Argon2id password hashing, short-lived access tokens with rotating refresh tokens, per-resource ownership enforcement, administrators cannot see private student content by default, validated and sandboxed AI output, encrypted Google tokens, validated and scanned uploads. See [Security](docs/17-security.md), [Privacy](docs/18-privacy.md) and [SECURITY.md](SECURITY.md).

## 12. Contributing

Read [CLAUDE.md](CLAUDE.md) and [CONTRIBUTING.md](CONTRIBUTING.md). Major architectural changes require an ADR and approval before implementation.

## 13. License

Proprietary, all rights reserved for now. See [LICENSE](LICENSE) and the open decision in the [architecture review](docs/25-architecture-review.md).
