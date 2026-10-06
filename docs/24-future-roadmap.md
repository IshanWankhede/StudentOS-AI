# 24. Future Roadmap

**Purpose:** document post-MVP and future capabilities and the architectural hooks that keep them possible. **Scope:** direction, not commitment. Each item needs its own design review and, where it changes architecture, an ADR.

See also: [Product Requirements](01-product-requirements.md) · [Development Roadmap](23-development-roadmap.md) · [Recommendation Engine](12-recommendation-engine.md) · [Architecture Review](25-architecture-review.md) · [Privacy](18-privacy.md)

## 1. Post-MVP

| Item | Description | Prerequisites and notes |
|---|---|---|
| Advanced recommendations | Contextual bandits over recommendation kinds and timing, richer personalisation | Feedback data volume; offline evaluation; keep deterministic fallback |
| YouTube assistant | Analyse a video: summary, key points, chapters, save to resources | Verify YouTube API terms and quota; transcript source must be compliant (no unofficial scraping); SSRF-safe fetcher; Video Analysis Service in the AI Gateway |
| Hackathon recommendation engine | Suggest hackathons by interests, skills and calendar | Needs a licensed or permitted data source (review M-09); user-submitted listings as interim |
| AI news | Personalised study/career news | Source licensing and copyright (review M-10); ingestion jobs; embeddings against interests; news tables are global content |
| Gamification | XP, streaks, achievements | Designed in [doc 15](15-gamification.md), roadmap Phase 17 |
| Advanced analytics | Trends, subject comparisons, time-of-day patterns, planning-quality metrics | Aggregation tables; strictly study behaviour, not health |
| Google Drive | Per-file import via Picker | `drive.file` scope; verification; validated upload pipeline |
| Email notifications | Deadline and digest emails with unsubscribe | Reuses email adapter; preference model exists |
| Advanced exam planner | Spaced repetition, adaptive confidence, multi-exam balancing, mock-test analytics | See [doc 11](11-exam-planning-engine.md#6-post-mvp-advanced-exam-planner) |
| Adaptive planning | Engines learn availability patterns and per-subject pace | Calibration already collected; keep engine deterministic with learned parameters stored explicitly |
| PWA and web push | Installable app, push notifications | Service worker and push provider decision |
| Distributed rate limiting | Redis-based counters | Only if multi-replica limits prove insufficient ([ADR 009](adr/009-background-jobs.md)) |

## 2. Future

| Item | Description | Architectural hooks and considerations |
|---|---|---|
| Android digital wellbeing | Screen-time-aware focus support | Requires a native client and OS permissions; **separate privacy design, consent and legal review**; must not diagnose; separate ADR |
| Advanced ML recommender | Trained models on aggregated, consented data | Feature store from analytics tables; privacy review; model serving separate from the monolith if needed |
| Voice assistant | Hands-free planning and timers | Speech provider abstraction in the AI Gateway; privacy of audio |
| Teacher accounts | Teachers view and assign coursework | New `teacher` role, organisation model, explicit student sharing consent; changes the ownership model, so requires an ADR |
| College dashboards | Institution-level aggregates | Tenant/organisation entity, aggregate-only access with k-anonymity thresholds; contractual data processing terms |
| Social features | Study groups, shared plans, leaderboards | Sharing and permissions model; moderation; minors and harassment considerations |
| AI agents | Agents that act on the student's behalf (reschedule, email) | Requires scoped tool permissions, per-action user confirmation, audit trail; extends, never bypasses, the validation pipeline |
| Mobile application | Native/React Native clients | Public API versioning already in place; token strategy for mobile (PKCE, secure storage) |
| Cross-college analytics | Benchmarking across institutions | Strong anonymisation, legal agreements, opt-in |

## 3. Extraction Candidates

If scale demands, extract behind existing public interfaces: AI processing (`ai`), sync workers (`google`), analytics aggregation. The modular boundaries and job-based async processing make this a deployment change rather than a redesign ([ADR 001](adr/001-modular-monolith.md#future-migration-path)).
