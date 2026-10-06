# ADR 006: Authentication Strategy

**Status:** Accepted (architectural recommendation within the specification's JWT/Google constraints) · **Date:** 2026-10-05 · **Related:** [Authentication](../07-authentication.md), [Security](../17-security.md)

# Context

The product needs email/password and Google login, a browser SPA client, a possible future mobile client, and strong protection of student data. The specification lists `JWT_SECRET`, `JWT_ALGORITHM` and `ACCESS_TOKEN_EXPIRE_MINUTES`.

# Problem

Choose token and session design that limits damage from XSS, CSRF and token theft while keeping the API stateless for normal calls.

# Options Considered

1. **Long-lived JWT in localStorage.**
2. **Server-side sessions with a session cookie** for everything.
3. **Short-lived JWT access token (memory) + rotating opaque refresh token in an HttpOnly cookie.**
4. Third-party identity provider (Auth0, Clerk, Cognito).

# Decision

Option 3. Access JWT: 15 minutes, `HS256` with `JWT_SECRET`, `kid` support for rotation, kept in browser memory. Refresh token: opaque 256-bit, stored hashed, rotated on every use with reuse detection revoking the family, HttpOnly Secure SameSite=Strict cookie scoped to `/api/v1/auth`, 30-day absolute / 14-day idle. Passwords hashed with Argon2id. Google sign-in via OIDC authorization code + PKCE, linked on `sub`. TOTP MFA required for admins (Phase 19).

# Reasoning

Memory-only access tokens are not readable from storage by injected scripts at rest and expire quickly; the refresh cookie is invisible to JavaScript. Rotation with reuse detection limits stolen-refresh-token value. Server-side session state exists only for refresh tokens, so sessions can be listed and revoked. HS256 is acceptable because a single service signs and verifies.

# Trade-offs

More moving parts than a simple session cookie; page reload requires a refresh call; CSRF defence must be maintained on cookie endpoints; same-site deployment of frontend and API is required; a symmetric secret means every verifier can also forge tokens.

# Consequences

Frontend implements silent refresh with single-flight; edge must route `/api` on the same site; email provider is needed in Phase 2 for verification and reset; audit and throttling tables are required.

# Future Migration Path

If verification moves to multiple services or third parties, switch to asymmetric signing (EdDSA/RS256) with a JWKS endpoint using the existing `kid` mechanism. For a mobile client, use the same refresh/rotation logic with tokens in secure OS storage and PKCE-based Google login. If requirements grow (SSO, enterprise), adopt an external identity provider and map `sub` to `auth_identities`.
