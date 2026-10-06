# Security Policy

## Security Principles

Least privilege, defense in depth, secure by default, ownership enforced by construction, never trust external input (users, Google, AI, uploaded files), minimise and protect student data. Full design: [Security](docs/17-security.md), [Privacy](docs/18-privacy.md), [Authentication](docs/07-authentication.md), [Authorization](docs/08-authorization.md).

## Reporting a Vulnerability

Use **GitHub Private Vulnerability Reporting** (Security tab → "Report a vulnerability") on this repository. Do not open public issues for security problems.

| Step | Target |
|---|---|
| Acknowledge report | within 3 business days |
| Initial triage and severity | within 7 days |
| Fix or mitigation, Critical/High | as soon as possible, target 14 days |
| Fix or mitigation, Medium/Low | next scheduled release |
| Public disclosure | coordinated with the reporter after a fix ships |

Include affected endpoint or component, reproduction steps, impact, and any proof-of-concept. Good-faith research that avoids data destruction, privacy violations and service degradation will not be pursued.

## Secret Management

No secrets in the repository; `.env` is git-ignored and `.env.example` holds placeholders only. Staging/production secrets live in the platform secret manager and are injected at runtime. CI runs secret scanning (gitleaks). Leaked secrets are rotated immediately. Google tokens are encrypted at rest with a rotatable key (`TOKEN_ENCRYPTION_KEY_ID`).

## Authentication

Argon2id password hashing; 15-minute access JWTs; opaque, hashed, rotating refresh tokens with reuse detection in an HttpOnly Secure cookie; Google sign-in via OIDC with PKCE, state and nonce; login throttling; MFA required for administrators (Phase 19). See [Authentication](docs/07-authentication.md).

## Authorization

Every user-owned resource is scoped to its owner in the repository layer; cross-user access returns 404; administrators see aggregates and metadata only; break-glass access requires user consent, expires, and is audited. See [Authorization](docs/08-authorization.md).

## Data Protection

TLS everywhere, encryption at rest for database and object storage, signed short-lived download URLs, data export and deletion supported, AI payloads retained briefly and encrypted. See [Privacy](docs/18-privacy.md).

## Dependency Security

Lockfiles committed; Dependabot or Renovate enabled; `pip-audit`, `npm audit`, Trivy image scan and CodeQL run in CI; new dependencies require justification in the PR.

## File Upload Security

Extension allowlist, magic-byte MIME sniffing, size limits, no active content (SVG, macro-enabled Office files, executables), malware scan before availability, private bucket, `Content-Disposition: attachment`, `nosniff`. See [File Storage](docs/16-file-storage.md).

## AI Security

Prompt-injection-aware context building, user and external text treated as data, no tool access to mutate data, schema + business validation of all output, quotas and budget breaker, PII minimisation toward providers. See [AI Architecture](docs/09-ai-architecture.md).

## OAuth Security

Authorization code flow with PKCE, exact redirect URI matching, state and nonce verification, least-privilege scopes requested incrementally, revocation on disconnect and account deletion. See [Google Integrations](docs/13-google-integrations.md).

## Incident Response

1. **Detect:** alerts and reports ([Observability](docs/20-observability.md)).
2. **Contain:** revoke sessions, rotate secrets, disable affected integration or feature flag.
3. **Assess:** scope using audit logs and request IDs.
4. **Eradicate and recover:** patch, redeploy, restore from backup if needed.
5. **Notify:** affected users and authorities where law requires (verify obligations with counsel).
6. **Review:** blameless post-mortem within 5 business days; update docs, tests and alerts.
