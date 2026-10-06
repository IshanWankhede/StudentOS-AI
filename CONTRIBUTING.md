# Contributing to StudentOS AI

**Purpose:** how to propose, build, review and merge changes. Applies to humans and AI agents. Read [CLAUDE.md](CLAUDE.md) first.

## Branch Strategy

Trunk-based development. `main` is always releasable and protected (PR required, green CI, at least one approving review, no force-push). Branch names: `feat/<scope>-<summary>`, `fix/…`, `docs/…`, `chore/…`, `refactor/…`. Branches live days, not weeks. Releases are tagged from `main`.

## Commit Conventions

[Conventional Commits](https://www.conventionalcommits.org/): `type(scope): summary`. Types: `feat, fix, docs, test, refactor, perf, chore, ci, build, security`. Scope is a backend module or `frontend`/`infra`/`docs`. Breaking changes use `!` and a `BREAKING CHANGE:` footer. Never commit secrets; if one is committed, rotate it immediately, because history rewrites do not undo exposure.

## Pull Requests

Keep PRs small and focused on one vertical slice or one concern. The description states: what and why, linked roadmap phase, screenshots for UI, migration notes, rollout/rollback notes, and the checklist below.

- [ ] Migration included and reversible or expand/contract safe
- [ ] API spec, DB design and relevant docs updated
- [ ] Validation and authorization implemented
- [ ] Authorization tests (cross-user access denied)
- [ ] Unit/integration tests; frontend tests for UI changes
- [ ] No secrets, no PII in logs, no new unreviewed dependency
- [ ] CHANGELOG updated

## Code Review

Reviewers check: module boundaries (no cross-module internals), layering, ownership scoping on every query, input and output validation, AI output validation, error handling, test quality, docs, and dependency justification. Authors respond to every comment; reviewers approve only when CI is green. Security-sensitive changes (auth, tokens, uploads, Google, AI prompts, admin) require a second reviewer.

## Testing Requirements

`make lint typecheck test` must pass locally; CI runs unit, integration, authorization, API contract, frontend and security checks ([Testing Strategy](docs/19-testing-strategy.md)). Bug fixes include a regression test.

## Documentation Requirements

Docs are part of the product. A change is incomplete if it alters behaviour documented in `docs/` without updating it. Use relative links and keep Mermaid diagrams truthful.

## Architecture Changes and ADRs

Any major architectural change follows the protocol in [CLAUDE.md §5](CLAUDE.md#5-architectural-change-protocol-most-important-rule). ADR files in `docs/adr/` use the sections Context, Problem, Options Considered, Decision, Reasoning, Trade-offs, Consequences, Future Migration Path. Supersede, never delete: mark old ADRs `Superseded by NNN`.

## Security Rules

Follow [Security](docs/17-security.md) and [SECURITY.md](SECURITY.md). Report vulnerabilities privately, never in public issues.
