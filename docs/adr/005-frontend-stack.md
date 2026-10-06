# ADR 005: Frontend Stack

**Status:** Accepted (core libraries specification-mandated; supporting choices are architectural recommendations) · **Date:** 2026-10-05 · **Related:** [Frontend Architecture](../03-frontend-architecture.md)

# Context

The product is an authenticated, highly interactive app (planner, timers, forms, dashboards) with no SEO needs for the app itself.

# Problem

Choose the SPA stack and state-management rules.

# Options Considered

1. **React + TypeScript + Vite** (SPA) with TanStack Query and Zustand.
2. Next.js (SSR/RSC).
3. Vue/Nuxt or Svelte/SvelteKit.
4. Redux Toolkit + RTK Query for state.

# Decision

React, strict TypeScript, Vite, React Router, TanStack Query for server state, Zustand for client-only state, Tailwind CSS with accessible primitives, React Hook Form + Zod, Recharts, Vitest/Testing Library/MSW, Playwright. API client generated from OpenAPI. Rule: server data is never copied into Zustand.

# Reasoning

An SPA behind authentication gains little from SSR and avoids a second server runtime. TanStack Query handles caching, polling (AI requests), optimistic updates and invalidation. Zustand is small and sufficient for token, UI and timer-render state.

# Trade-offs

No server rendering for marketing pages (a separate static site can serve those); two state libraries require the discipline in the rule above; calendar component library choice (license, bundle size) remains open until Phase 4.

# Consequences

Static assets served from a CDN; single build artifact configured at runtime; access token kept in memory with refresh via cookie ([ADR 006](006-authentication-strategy.md)). Frontend features mirror backend modules.

# Future Migration Path

If SEO or first-load performance for public pages becomes important, add a separate marketing site or adopt a meta-framework for those routes while keeping the SPA. A mobile app can reuse the generated API client types and TanStack Query patterns (React Native).
