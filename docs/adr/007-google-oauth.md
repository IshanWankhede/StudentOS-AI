# ADR 007: Google OAuth and Integrations

**Status:** Accepted (specification-mandated Google use; scope and storage choices are architectural recommendations, subject to verification) · **Date:** 2026-10-05 · **Related:** [Google Integrations](../13-google-integrations.md), [Privacy](../18-privacy.md)

# Context

Students authenticate with Google and expect Calendar and Classroom integration. Calendar and Classroom scopes are sensitive, Google applies verification and data-use policies, and refresh tokens are high-value secrets.

# Problem

Decide how to authenticate with Google, request permissions, store tokens, and design sync to minimise risk and review burden.

# Options Considered

1. Single broad consent at sign-in (all scopes up front).
2. **Sign-in with identity scopes only, then incremental per-capability authorization.**
3. Service-account or domain-wide delegation (not applicable to personal student accounts).
4. Avoid Google integrations entirely.

# Decision

Option 2. Sign-in uses OIDC (`openid email profile`). Calendar, Classroom and Drive are separate capabilities connected on demand with `include_granted_scopes`. Prefer narrow scopes: a dedicated "StudentOS" calendar with `calendar.app.created` plus `calendar.freebusy` (fallback `calendar.events`), Classroom read-only student scopes, Drive only `drive.file` via Picker. All scope names and behaviours are **[VERIFY]** against official documentation before implementation. Tokens are stored encrypted (AES-256-GCM, rotatable key id; KMS envelope in production). Calendar sync is one-way push plus free/busy pull; Classroom is read-only pull. Disconnect revokes at Google and deletes local tokens and mappings.

# Reasoning

Incremental consent matches user expectations and reduces drop-off and review scope. Writing only to a dedicated calendar prevents damage to the user's real events. One-way sync avoids conflict resolution complexity in MVP.

# Trade-offs

Multiple consent screens; narrow scopes may be unavailable or insufficient (fallback costs broader permission); verification lead time can delay launch; Workspace administrators may block access; polling instead of webhooks increases latency and quota use.

# Consequences

A token vault module, sync-run history and `needs_reauth` state exist; production OAuth consent screen and verification must be started at Phase 13; Google user data used in AI prompts is constrained by Limited Use policy; separate Google Cloud projects per environment.

# Future Migration Path

Add Calendar push notifications (webhooks) and two-way edits once one-way sync is stable. Add Drive import via Picker. Move token encryption to a managed KMS with per-user data keys. If Google policy blocks AI use of this data, route those features to deterministic paths without LLM involvement.
