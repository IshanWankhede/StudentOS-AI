# ADR 008: AI Provider Abstraction

**Status:** Accepted (specification-mandated) · **Date:** 2026-10-05 · **Related:** [AI Architecture](../09-ai-architecture.md)

# Context

Multiple AI features need LLM and embedding calls. Provider APIs, models, prices and terms change often, and student data privacy depends on provider terms.

# Problem

Avoid coupling the application to one provider while still using structured outputs, enforcing validation, quotas, logging and privacy.

# Options Considered

1. Call a provider SDK directly from each feature.
2. Use a third-party multi-provider library throughout the codebase.
3. **Internal AI Gateway** with `LLMProvider`/`EmbeddingProvider` interfaces and thin adapters.

# Decision

Option 3. Only the `ai` module imports provider SDKs (enforced by import-linter). Features call `AIGateway`, requesting a logical tier (`fast`, `standard`, optional `reasoning`) and a Pydantic output schema. The gateway handles prompt versioning, context minimisation and pseudonymisation, retries and fallback models, schema and business validation, cost tracking, quotas, logging, and persistence of proposals. A deterministic `FakeProvider` is the default in development and CI. No model tools or autonomous actions are exposed.

# Reasoning

A single seam makes provider switching, cost control, privacy controls and safety validation uniform. Logical tiers decouple features from model names. Structured, validated proposals reduce hallucination impact.

# Trade-offs

An abstraction can hide provider-specific capabilities (native structured output, caching); adapters must be maintained; lowest-common-denominator risk is mitigated by capability flags per adapter.

# Consequences

All AI usage is auditable and quota-controlled. Prompt files are versioned in the repository. A golden-set evaluation is needed to compare providers. Provider selection and data terms are an owner decision before production use (review D-03).

# Future Migration Path

Add providers by writing an adapter and mapping tiers. Add a fallback chain across providers once two are in use. Self-hosted or on-device models can be added behind the same interface. If AI processing needs isolation, extract it as a service behind the gateway interface ([ADR 001](001-modular-monolith.md#future-migration-path)).
