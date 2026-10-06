# 25. Architecture Review

**Purpose:** a lead-architect review of the specification and the resulting design: gaps, contradictions, risks and recommended changes. **Scope:** whole project. Items marked *architectural recommendation* are my proposals and need owner approval; they are not silent requirement changes.

See also: [Product Requirements](01-product-requirements.md) · [System Architecture](02-system-architecture.md) · [Security](17-security.md) · [Privacy](18-privacy.md) · [Google Integrations](13-google-integrations.md) · [Development Roadmap](23-development-roadmap.md)

## 1. Missing Requirements

| ID | Gap | Impact | Resolution in this design |
|---|---|---|---|
| M-01 | Reminders and notifications are not in the MVP list, yet deadlines and sessions are the core value | Productivity app without reminders underdelivers | In-app notifications added to MVP (Phase 16); email stays post-MVP |
| M-02 | Timezone and locale handling unspecified | DST and travel bugs in scheduling | UTC storage, per-user IANA timezone, DST-safe Constraint Engine |
| M-03 | Academic calendar (terms, holidays, exam weeks) absent | Subjects and timetable need term bounds | `academic_terms` and `timetable_entries.valid_from/valid_until`; holidays deferred |
| M-04 | File storage and Resources API required but not in the MVP list | Hackathon/project files unusable | File uploads for projects/hackathons in MVP; resource library with embeddings in Phase 15 |
| M-05 | Transactional email (verification, reset) is needed from Phase 2 although email notifications are post-MVP | Email auth cannot work without it | Email provider adapter in Phase 2; env vars added |
| M-06 | Dashboard and Timetable have no roadmap phase | Orphaned MVP features | Timetable in Phase 4, dashboard grows in Phases 4, 9, 15 |
| M-07 | Source and quality of time estimates unspecified | Planning accuracy | Estimate source tracked; calibration factor from history; default 60 min flagged |
| M-08 | Minimum age, consent model, and applicable law (GDPR, India DPDP, minors) undefined | Legal exposure | Documented technical consent model; **owner decision and legal review required** |
| M-09 | No data source for hackathon recommendations | Post-MVP feature may be infeasible | MVP is user-entered only; source decision deferred |
| M-10 | AI news source, licensing and copyright undefined | Legal and quality risk | Deferred; require licensed feeds/RSS terms before building |
| M-11 | YouTube assistant needs transcript access and API quota | ToS and cost risk | Deferred; compliant transcript source required; SSRF-safe fetcher |
| M-12 | Account recovery, MFA, admin bootstrap undefined | Account takeover and admin risk | Reset flow in Phase 2; admin MFA Phase 19; admin promotion manual and audited |
| M-13 | Accessibility, i18n, browser/mobile support, PWA undefined | Reach and compliance | WCAG 2.1 AA target, responsive web, PWA post-MVP; i18n-ready strings recommended |
| M-14 | Multi-device behaviour of the Pomodoro timer undefined | Inconsistent timers | Server-authoritative timestamps; one active session per user |
| M-15 | Soft vs hard deadlines and recurring tasks not specified | Planning nuance | Single due date in MVP; recurring tasks out of scope; revisit post-MVP |
| M-16 | AI cost model, monetisation and quotas undefined | Runaway cost | Per-user quotas and global budget breaker |
| M-17 | Admin scope undefined | Privacy conflict | Minimal admin: aggregates, suspension, jobs, audit, flags, break-glass |
| M-18 | Hosting target and budget undefined | Deployment choices | Provider-agnostic design with a managed-PaaS recommendation |
| M-19 | License and copyright holder unspecified | Open-source/IP status | LICENSE is all-rights-reserved placeholder default; **owner decision** |
| M-20 | LLM provider and data-processing terms undefined | Privacy and Google Limited Use compliance | Provider abstraction; provider choice and contract review required before Phase 6 production use |

## 2. Contradictions

| ID | Conflict | Resolution |
|---|---|---|
| C-01 | "Never expose private data to admins" vs an Admin module and support needs | Admin is aggregate/metadata only; consented, time-boxed break-glass ([08](08-authorization.md)) |
| C-02 | "Basic Classroom sync" in MVP vs Classroom access limits (Workspace admin restrictions, OAuth verification lead time) | Keep in MVP as Phase 14 but treat as schedule risk; start verification early; degrade gracefully |
| C-03 | "LLM must not control scheduling" vs "AI roadmap/planning assistant" | AI breaks down and explains; engines decide ([10](10-task-planning-engine.md)) |
| C-04 | Zustand and TanStack Query overlap | Query for server state, Zustand only for client-only state ([03](03-frontend-architecture.md)) |
| C-05 | Testing as Phase 20 vs Definition of Done requiring tests per feature | Testing is continuous; Phase 20 is hardening |
| C-06 | "API should return quickly" vs synchronous AI validation flow | Async AI with 202 + polling; validation happens in the worker |
| C-07 | Background worker required with no queue technology; "avoid unnecessary dependencies" vs a typical Redis stack | PostgreSQL-backed queue (Procrastinate) ([ADR 009](adr/009-background-jobs.md)) |
| C-08 | Upload path through backend validation vs the usual direct-to-storage best practice | Backend-proxied uploads in MVP with 25 MiB cap; migration path documented ([ADR 010](adr/010-object-storage.md)) |
| C-09 | `JWT_SECRET`/`JWT_ALGORITHM` (symmetric) vs possible future multi-service verification | HS256 acceptable in a monolith; `kid` support; migration to asymmetric keys documented ([ADR 006](adr/006-authentication-strategy.md)) |
| C-10 | Email auth and Google login in MVP, but the `.env.example` list has no email provider variables | Variables added |
| C-11 | Roadmap phases 16–17 sit before hardening, but gamification is post-MVP | Release boundary explicit: Phase 17 is post-MVP |
| C-12 | pgvector mandated while MVP embedding use is thin | Extension installed Phase 1; first use Phase 15 (resources/tasks similarity) |
| C-13 | Google Drive documented but post-MVP, and broad Drive scopes are heavily restricted | Only `drive.file` + Picker, post-MVP |
| C-14 | "Do not invent Google API behaviour" vs "document required scopes" | All uncertain facts marked **[VERIFY]** |

## 3. Security Risks

| ID | Risk | Mitigation |
|---|---|---|
| S-01 | IDOR across ~25 owned resource types (the most likely real-world bug class) | Ownership by construction, composite FKs, generated authorization matrix, RLS (Phase 19) |
| S-02 | XSS exposing the in-memory access token | Strict CSP, sanitised Markdown, 15-minute tokens, refresh cookie not script-readable |
| S-03 | CSRF on cookie endpoints | SameSite=Strict, Path scope, custom header, Origin check; same-site deployment is a hard requirement |
| S-04 | Account pre-hijacking and unsafe Google account linking | Link by `sub`; unverified local account invalidated on Google link |
| S-05 | Google token theft from the database | Encryption at rest with rotation, least scopes, revocation |
| S-06 | Malicious uploads (polyglots, zip bombs, SVG XSS, path traversal) | Allowlist, magic bytes, scan, opaque keys, attachment disposition, private bucket |
| S-07 | Prompt injection through Classroom text, syllabus text, files | Untrusted-data framing, no tools, validated output, user-accepted proposals |
| S-08 | SSRF via user-supplied URLs | MVP never fetches them; hardened fetcher later |
| S-09 | Admin overreach and insider threat | Aggregates only, MFA, audit, break-glass with consent |
| S-10 | Weak rate limiting with multiple replicas and no shared store | DB-backed auth/AI limits plus edge limits; Redis if needed |
| S-11 | `JWT_SECRET` compromise allows token forgery | Strong secret, rotation with `kid`, secret manager, short lifetimes |
| S-12 | Email/user enumeration | Uniform responses and timing |
| S-13 | Mass assignment and parameter tampering | `extra=forbid`, no `user_id` in input |
| S-14 | Supply-chain compromise | Lockfiles, audits, CodeQL, Trivy, review of new dependencies |
| S-15 | Secrets or PII leaking into logs | Redaction processor and tests |

## 4. Privacy Risks

| ID | Risk | Mitigation |
|---|---|---|
| P-01 | Student data sent to LLM providers | Minimisation, pseudonymous refs, user opt-out, contract terms, 14-day payload TTL |
| P-02 | Google data used for AI conflicts with Limited Use policy | Verify policy; no training; disclosure; minimal fields |
| P-03 | Analytics interpreted as mental-health inference or used coercively | Study-behaviour-only definition; copy blocklist; no diagnosis; no sharing with third parties |
| P-04 | Deleted data persisting in backups or logs | ≤ 35-day backup retention; re-apply deletions on restore; content-free logs |
| P-05 | Third-party personal data (teammate names) entered by students | Warn users; minimal fields; included in export/deletion |
| P-06 | Minors using the service | Define minimum age; legal review (M-08) |
| P-07 | Cross-border transfers (LLM provider, hosting region) | Region selection and provider terms decision; document in privacy policy |
| P-08 | Incomplete exports or slow deletion | Module-by-module export registry test; deletion job idempotency and alerts |
| P-09 | Consent drift as features change | Versioned consents, re-prompt on purpose change |

## 5. Scalability Risks

| ID | Risk | Mitigation |
|---|---|---|
| SC-01 | Deadline clustering (exam season, midnight due dates) causes correlated load | Rate smoothing, queue priorities, caching of dashboards |
| SC-02 | Replan storms from bulk imports or calendar changes | Per-user debounce and coalescing |
| SC-03 | PostgreSQL as queue under high job rates | Adequate at expected scale; migration to Redis/SQS behind the same job interface |
| SC-04 | Google quotas with many users | Jittered polling, backoff, per-user locks, adaptive frequency |
| SC-05 | pgvector memory/index cost | Per-user exact scans; HNSW only when measured |
| SC-06 | Notification scheduler scans | Partial index and batched `SKIP LOCKED` |
| SC-07 | Connection exhaustion with async workers and replicas | Pool sizing, PgBouncer in stage 2 (transaction-local settings for RLS) |
| SC-08 | Analytics aggregation growth | Daily rollup table, rebuildable, partition later |
| SC-09 | Storage growth | Quotas, lifecycle rules for exports |
| SC-10 | Single primary database | Vertical scale, read replica, then targeted extraction |

## 6. AI Risks

| ID | Risk | Mitigation |
|---|---|---|
| A-01 | Hallucinated subtasks, topics, dependencies | Grounding, business validation, human review before persistence |
| A-02 | Invalid structured output | Schema validation, repair retries, failure state, manual fallback |
| A-03 | Bad recommendations (irrelevant or demotivating) | Deterministic generators, reason required, feedback loop, cooldowns, neutral wording |
| A-04 | Incorrect effort estimates | Labelled as AI, calibration from actuals, ranges in UI |
| A-05 | Prompt injection | See S-07 |
| A-06 | Excessive cost or abuse | Quotas, concurrency caps, global budget breaker, dedupe |
| A-07 | Privacy leakage to providers | See P-01/P-02 |
| A-08 | Provider lock-in or deprecation | Gateway abstraction, logical tiers, fake provider, golden-set evaluation for switching |
| A-09 | Quality regressions after model/prompt changes | Prompt versioning, nightly golden-set evaluation |
| A-10 | Overreliance on AI plans | Explanations, editability, deterministic engine remains source of truth |

## 7. Google API Risks

| ID | Risk | Mitigation |
|---|---|---|
| G-01 | OAuth scope breadth and verification lead time | Narrow scopes, incremental authorization, start verification at Phase 13 |
| G-02 | Refresh token expiry/revocation (including Testing-mode short expiry) | `needs_reauth` flow, notifications, production-status project for real users |
| G-03 | Quotas and rate limits | Backoff with jitter, per-user locks, lower polling under pressure |
| G-04 | Permission changes by users or Workspace admins | Granted-scope check on every connect, graceful degradation, clear messaging |
| G-05 | API availability or behaviour changes | Response validation, contract tests with a fake client, version pinning of client libs |
| G-06 | Sync conflicts (remote edits/deletes) | StudentOS is source of truth for pushed events; link statuses; Classroom override rules |
| G-07 | Policy compliance (Limited Use, AI training ban, disclosure) | Design in [13 §7](13-google-integrations.md#7-google-data-and-ai); legal verification |
| G-08 | Workspace for Education restrictions on third-party apps | Detect and explain; keep manual task entry as the baseline |

## 8. Implementation Risks

| ID | Risk | Mitigation |
|---|---|---|
| I-01 | Module boundary erosion in a modular monolith | import-linter contracts in CI, `public.py` rule |
| I-02 | Async SQLAlchemy pitfalls (lazy loading, session scope) | Documented rules, integration tests, eager loading |
| I-03 | Procrastinate maturity and API fit | Verify current version and features before Phase 6; thin job interface so a swap is cheap |
| I-04 | DST/timezone scheduling bugs | Property tests across DST boundaries, injected clock |
| I-05 | Composite ownership FKs add ORM complexity | Helper base classes and tests; accepted for safety |
| I-06 | OpenAPI/client drift | CI contract check |
| I-07 | Scope creep across 22 phases | Strict phase gates and MVP boundary |
| I-08 | Small team capacity | Lean defaults, vertical slices, defer post-MVP items |
| I-09 | Embedding dimension lock-in (`vector(1024)`) | Model must support 1024 dims; re-embed migration documented |
| I-10 | Pomodoro timer drift and background-tab throttling | Server timestamps, visibility re-sync |
| I-11 | pgvector or `btree_gist` availability on managed Postgres | Verify at provider selection |
| I-12 | Exclusion constraint on planned sessions causing conflicts during replans | Replan in a single transaction deleting unlocked future sessions first |

## 9. Recommended Changes

| ID | Recommendation | Where recorded |
|---|---|---|
| R-01 | Use a PostgreSQL-backed job queue instead of adding Redis | [ADR 009](adr/009-background-jobs.md) |
| R-02 | Add in-app notifications to MVP | [14](14-notification-system.md) |
| R-03 | Add transactional email adapter at Phase 2 | [07](07-authentication.md) |
| R-04 | Async AI with proposals and acceptance | [09](09-ai-architecture.md) |
| R-05 | Dedicated "StudentOS" Google calendar with narrow scopes | [13](13-google-integrations.md), [ADR 007](adr/007-google-oauth.md) |
| R-06 | Composite ownership FKs and optional RLS | [05](05-database-design.md), [08](08-authorization.md) |
| R-07 | Break-glass admin access model | [08](08-authorization.md) |
| R-08 | Store text + CHECK instead of PG enums; UUIDv7 | [05](05-database-design.md) |
| R-09 | Backend-proxied uploads with scanning and quarantine | [16](16-file-storage.md), [ADR 010](adr/010-object-storage.md) |
| R-10 | Quotas and a global AI budget breaker | [09](09-ai-architecture.md) |
| R-11 | Start Google OAuth verification at Phase 13 | [23](23-development-roadmap.md) |
| R-12 | Continuous testing; Phase 20 as hardening | [19](19-testing-strategy.md) |

## 10. Decisions Needing Owner Approval Before Implementation

| ID | Decision | My recommendation |
|---|---|---|
| D-01 | Background jobs on PostgreSQL (Procrastinate) rather than Redis | Approve |
| D-02 | Minimum user age and consent model | 18+ initially; verify with counsel |
| D-03 | LLM provider(s) and contract terms | Choose one provider with no-training terms; keep abstraction |
| D-04 | Hosting platform and region | Managed container platform with managed PostgreSQL supporting pgvector |
| D-05 | License | Keep all-rights-reserved until decided |
| D-06 | Exact numeric targets in the NFR table and AI quotas | Accept as defaults, tune after beta |
| D-07 | Email provider | Any provider with SPF/DKIM support and an adapter |
| D-08 | Whether in-app notifications are MVP | Yes |

## 11. Items To Verify Before Implementation

Google: current scope names and sensitivity classification (Calendar narrow scopes, Classroom student scopes), refresh-token behaviour in Testing vs Production, Classroom state names, Calendar event-id rules and sync-token behaviour, verification requirements, Limited Use text. Libraries: Procrastinate current features and compatibility with the chosen Python/PostgreSQL versions, async SQLAlchemy driver choice, pgvector version and managed-provider support, password-hash parameter benchmarks on target hardware. Legal: privacy law applicability, minors, cross-border transfer, provider data-processing terms. These are also flagged inline in the relevant documents.

## 12. Documentation Consistency Review

Checks performed on the generated set: required file inventory, relative link resolution, Mermaid block count and fence balance, forbidden placeholder terms, and cross-document alignment of enums, table names, endpoints and phases. Results: all 43 required files exist; 385 relative links and anchors resolve; all 32 Mermaid diagrams parse with the Mermaid 11 parser; all 10 ADRs contain the 8 mandated sections; no placeholder terms (TODO/TBD) remain. Known limits of this review: the API specification gives per-endpoint tables plus one worked example per module rather than a separate example for every endpoint, and Google API facts remain **[VERIFY]** items.
