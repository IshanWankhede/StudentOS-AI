# 17. Security

**Purpose:** consolidated security architecture and controls. **Scope:** application, API, data, uploads, AI, secrets, operations. Targets OWASP ASVS Level 2 as a guide.

See also: [Authentication](07-authentication.md) · [Authorization](08-authorization.md) · [Privacy](18-privacy.md) · [AI Architecture](09-ai-architecture.md) · [File Storage](16-file-storage.md) · [Google Integrations](13-google-integrations.md) · [SECURITY.md](../SECURITY.md)

## 1. Threat Model Summary

| Asset | Threats | Primary controls |
|---|---|---|
| Student data (tasks, notes, files, analytics) | IDOR, SQL injection, admin overreach, leakage in logs/AI | Ownership scoping, composite FKs, parameterised SQL, admin isolation, log redaction, AI minimisation |
| Accounts | Credential stuffing, phishing, token theft, pre-hijacking | Argon2id, throttling, short access tokens, rotating refresh tokens, verified-email linking, MFA for admin |
| Google tokens | Database leak, SSRF/exfil | Encryption at rest with rotation, least scopes, revoke on disconnect |
| Uploads | Malware, polyglot files, stored XSS, zip bombs, path traversal | Allowlist, magic bytes, scan, private bucket, attachment disposition, opaque keys |
| AI layer | Prompt injection, data exfiltration, cost abuse | Untrusted-data framing, schema+business validation, no tools, quotas, budget breaker |
| Availability | Abuse, expensive endpoints | Rate limits, quotas, timeouts, pagination caps |

## 2. Controls

| Area | Control |
|---|---|
| Passwords | Argon2id, breached-password check, no truncation ([07](07-authentication.md)) |
| OAuth | Code flow + PKCE, state, nonce, exact redirect URIs, least scopes, incremental consent ([13](13-google-integrations.md)) |
| Tokens | Access JWT 15 min in memory; refresh opaque, hashed, rotated, family revocation; Google tokens AES-256-GCM encrypted |
| Authorization | RBAC + ownership by construction, optional RLS ([08](08-authorization.md)) |
| CORS | Allowlist from `CORS_ALLOWED_ORIGINS` (exact origins, no wildcard with credentials); only needed methods/headers; preflight cached |
| CSRF | Only the refresh/logout cookie endpoints are cookie-authenticated: SameSite=Strict, Path scoped, custom header required, Origin check; all other calls use Authorization header |
| Input validation | Pydantic `extra=forbid`, length/range/enum limits, cross-field rules in services, UUID path params typed |
| SQL injection | SQLAlchemy parameterised queries only; raw SQL banned by lint rule except reviewed migrations |
| XSS | React escaping; Markdown rendered through a sanitising renderer; no `dangerouslySetInnerHTML` without sanitiser; strict CSP |
| Secure headers | `Strict-Transport-Security` (preload-ready), `Content-Security-Policy` (default-src 'self', no inline scripts, `frame-ancestors 'none'`, connect-src limited to API origin), `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` minimal, `Cross-Origin-Opener-Policy: same-origin` |
| HTTPS | TLS 1.2+ everywhere, HSTS, secure cookies, HTTP redirected at the edge |
| Secrets | Platform secret manager, never in repo/images/logs, per-environment, rotation runbook, gitleaks in CI |
| Audit logging | See §4 |
| Resource ownership | Enforced as in [08](08-authorization.md); 404 on foreign ids |
| Admin isolation | Aggregates only, MFA, break-glass with consent, audited ([08](08-authorization.md)) |
| Dependencies | Lockfiles, Dependabot/Renovate, `pip-audit`, `npm audit`, Trivy, CodeQL ([22](22-ci-cd.md)) |
| Error handling | No stack traces or SQL in responses; request id returned for support |

## 3. Rate Limiting

Layered, because there is no shared in-process store across replicas in MVP:

| Layer | Scope | Limits (architectural recommendation) |
|---|---|---|
| Edge/CDN | Per IP, all `/api` | 600 req/min; body size caps (25 MiB uploads, 1 MiB JSON) |
| Auth throttling (DB-backed `login_attempts`) | Per email and per IP | 5 failures / 15 min then progressive delay; register 5/hour/IP; forgot-password 3/hour/email |
| AI quotas (DB-backed `ai_usage_daily`) | Per user | 30 requests/day, 5/min, 3 concurrent |
| Expensive endpoints | Per user, application-level | `/resources/search` 30/min; `/recommendations/refresh` 1/10 min; manual sync 4/hour; exports 2/day |
| Google outbound | Per user + global | Token bucket in worker |

Introduce Redis for distributed counters only if measured need appears ([ADR 009](adr/009-background-jobs.md)).

## 4. Audit Logging

Logged in `audit_logs` (ids and action names only; never content or secrets): login success/failure, refresh reuse, logout-all, password change/reset, email change, MFA events, Google connect/disconnect, data export request/download, account deletion request/cancel/complete, admin actions (every `/admin` call), break-glass grant create/use/revoke, role/status changes, feature flag changes. Retention: 12 months; audit table is append-only for the application role.

## 5. SSRF and Outbound Requests

The server fetches external URLs only through a hardened fetcher: https only, DNS resolved then IP checked against private/loopback/link-local/metadata ranges (re-checked after redirects, max 3), timeouts, size cap, content-type allowlist, no cookies, dedicated egress. In MVP the server does **not** fetch student-supplied GitHub or resource URLs; the fetcher is introduced for post-MVP link previews and video analysis.

## 6. File Upload Security

See [File Storage](16-file-storage.md#3-validation-rules): allowlist, magic bytes, size/quota limits, scan before availability, no SVG/HTML/macro files, opaque keys, private bucket, signed 5-minute URLs, attachment disposition, nosniff.

## 7. AI Security

See [AI Architecture](09-ai-architecture.md#7-prompt-injection-and-untrusted-content): untrusted-data framing, no model tools, validated outputs, user-accepted proposals only, quotas, budget breaker, payload encryption with 14-day TTL.

## 8. Secure Development

Threat review for each new module, authorization test per endpoint, static analysis (Ruff security rules, CodeQL), dependency review, security checklist in PR template ([CONTRIBUTING.md](../CONTRIBUTING.md)), annual (and pre-launch) penetration test in roadmap Phase 19.

## 9. Operational Security

Least-privilege database roles (app role has no DDL, no access to other schemas; migration role separate; read-only analytics role if introduced), network isolation (database not publicly reachable), encrypted backups, MFA on cloud and GitHub accounts, branch protection, signed images, runtime secrets only via environment injection.

## 10. Failure Cases

| Case | Response |
|---|---|
| Secret leaked | Rotate immediately; invalidate sessions if `JWT_SECRET`; re-encrypt Google tokens if encryption key; incident process in [SECURITY.md](../SECURITY.md) |
| Suspected account takeover | Revoke families, force password reset, audit review |
| Malicious upload detected | Quarantine, notify user, record metadata, block hash |
| Google token compromise | Revoke at Google, delete, require reconnect |
