# Feature Specification: Dual AI Providers (Anthropic + Microsoft AI Foundry)

**Feature Branch**: `003-dual-ai-providers`

**Created**: 2026-07-27

**Status**: Draft

**Input**: User description: "in project @repostories/orbit we're using Anthropic connection only as an AI connection. We wanted to keep two providers at the same time, one of them Anthropic and one of them Microsoft AI Foundry connection."

## Clarifications

### Session 2026-07-27

- Q: When an org's selected provider fails, is cross-provider failover always allowed, or must it respect a data boundary? → A: Failover is on by default, but an organisation can be marked "strict" (no cross-provider failover) so residency-bound orgs never leave their selected provider.
- Q: Which failures trigger cross-provider failover (especially 429 rate-limits and content-policy refusals)? → A: Fail over on timeout / 5xx / connection-unreachable / 429 rate-limit; treat content-policy or safety refusals and other 4xx client errors as deterministic (no failover — degrade gracefully on the selected provider).
- Q: How fast must failover be — what bounds the added latency? → A: Declare the active provider unavailable after a bounded per-attempt wait (~20s target), attempt at most one cross-provider failover, so a failed-over request completes within roughly twice a normal call's latency.
- Q: How does Orbit authenticate to Microsoft AI Foundry? → A: Entra Managed Identity when running on Azure (keyless; any secret in Key Vault); key-based auth only as a documented development / non-Azure fallback.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - An organisation runs its AI on its selected provider (Priority: P1)

Both providers (Anthropic and Microsoft AI Foundry) are connected to the application at the
same time. Each organisation carries a setting that selects which provider serves its AI
requests. Every AI feature the organisation uses — email and journey generation, insights
Q&A, classification, event drafting, chat replies, and the rest — runs through that selected
provider. When an organisation is set to Foundry, the three task tiers (generation,
classification, reasoning) resolve to available Foundry model deployments so that every
feature works, not just some. Organisations that are not explicitly moved stay on Anthropic
with no change in behaviour.

**Why this priority**: This is the whole point of the request — being able to keep two
providers live and choose between them per organisation. Nothing else (failover, reporting)
matters until an organisation can actually run end-to-end on a chosen provider.

**Independent Test**: Set one test organisation to Foundry and another to Anthropic, then
exercise the same AI features in each. Confirm each organisation's requests are served by its
selected provider and produce valid results, while a third, untouched organisation continues
to run on Anthropic exactly as before.

**Acceptance Scenarios**:

1. **Given** an organisation set to Anthropic, **When** any AI feature is used, **Then** the request is served by Anthropic and behaviour is identical to today.
2. **Given** an organisation set to Foundry, **When** any of the AI features is used (across generation, classification, and reasoning tasks), **Then** the request is served by Foundry using an available model for that task tier and returns a valid result.
3. **Given** an existing organisation with no provider explicitly chosen, **When** the feature ships, **Then** it defaults to Anthropic and nothing about its AI behaviour changes.
4. **Given** two organisations set to different providers, **When** they use AI at the same time, **Then** each is served by its own provider with no cross-talk.

---

### User Story 2 - AI features keep working when a provider has an outage (Priority: P2)

When an organisation's selected provider fails on a transient or availability error (timeout,
5xx, rate limit, or the provider being unreachable), the same request is automatically retried
on the other connected provider so the feature still succeeds. The request only fails when
both providers fail, and even then it degrades gracefully rather than surfacing a hard error
to the user.

**Why this priority**: Keeping two providers live is only worth the effort if it buys
resilience. Failover turns "two connections" into "no single provider outage takes AI down,"
which is a primary reason to run both.

**Independent Test**: Force the selected provider to fail (simulate an outage/error) for a
test organisation, issue an AI request, and confirm the request completes via the other
provider. Then force both to fail and confirm the request degrades gracefully with no
user-facing 500.

**Acceptance Scenarios**:

1. **Given** an organisation whose selected provider returns a transient/availability error, **When** an AI request is made, **Then** the same request is retried on the other provider and succeeds.
2. **Given** a request that requires a validated/structured result, **When** failover to the other provider occurs, **Then** the same output guarantees (validation) still hold.
3. **Given** both providers are unavailable, **When** an AI request is made, **Then** the request fails gracefully (fallback/degraded response, never a user-facing 500) and the failure is recorded.
4. **Given** a deterministic failure that would fail on any provider (e.g. invalid input), **When** it occurs, **Then** the system does not pointlessly loop between providers.

---

### User Story 3 - Usage and cost stay governable across both providers (Priority: P3)

Every AI call records which provider (and model) served it, including whether a failover
occurred. Spend caps and cost tracking account for spend on both providers against the same
per-organisation limits, and operators can see a per-provider breakdown of usage and cost.

**Why this priority**: Once traffic can flow to either provider, cost governance and
attribution must not go blind. This protects against runaway spend and lets the team compare
providers, but it is not required for the first end-to-end provider switch to work.

**Independent Test**: Run AI on both providers for one organisation, then inspect usage
records and cost reporting; confirm each call is attributed to the provider that served it and
that the organisation's spend cap counts spend from both providers together.

**Acceptance Scenarios**:

1. **Given** AI calls served by different providers, **When** usage is inspected, **Then** each record identifies the serving provider and model.
2. **Given** an organisation with a spend cap, **When** it incurs cost on both providers, **Then** the cap counts the combined spend and pauses AI when breached.
3. **Given** a failover occurred, **When** the call is inspected, **Then** the record shows both the attempted and the serving provider.

---

### Edge Cases

- An organisation is set to a provider whose credentials are missing or misconfigured at the application level → the request fails gracefully for that organisation without affecting organisations on the working provider; the misconfiguration is visible to operators.
- The selected provider cannot serve a specific request even after tier mapping (e.g. a multimodal/image request to a text-only Foundry deployment) → the request either falls back to a provider that can serve it or degrades gracefully with a clear reason.
- An organisation's provider is changed while requests are in flight → in-flight requests complete on their original provider; the next request uses the new provider.
- Cost rate data is unknown for a newly served model (e.g. a Foundry/GPT-4o deployment) → usage logging still succeeds and the missing rate is surfaced rather than crashing the call.
- Failover must be bounded — a request must not bounce between providers indefinitely.
- A provider returns a content-policy / safety refusal (e.g. an Azure content-filter block) → treated as deterministic: the request is not failed over to the other provider and degrades gracefully, so failover never routes around a safety block.
- A strict organisation (failover disabled) hits an outage on its selected provider → the request degrades gracefully on that provider and is never retried on the other provider, so its data never crosses the provider boundary.
- Embeddings / knowledge-base retrieval are unaffected and continue on their current provider regardless of the selected LLM provider.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST maintain simultaneous, ready connections to two AI providers — Anthropic and Microsoft AI Foundry.
- **FR-002**: Each organisation MUST carry an AI-provider setting that selects which connected provider serves its AI requests.
- **FR-003**: Existing organisations MUST default to Anthropic, with no change to current AI behaviour when the feature ships (backward compatible).
- **FR-004**: All AI-backed features MUST route through the organisation's selected provider — the complete set, not a subset.
- **FR-005**: When Foundry is the selected provider, the generation, classification, and reasoning task tiers MUST resolve to available Foundry model deployments so that every AI feature functions.
- **FR-006**: When the selected provider fails on an availability or capacity error — a timeout, a 5xx, a connection/unreachable failure, or a 429 rate-limit — the system MUST automatically retry the same request on the other connected provider before returning a failure, unless the organisation is marked strict (see FR-016).
- **FR-007**: An AI request MUST fail only when both providers fail, and such failure MUST degrade gracefully (no user-facing 500), consistent with today's error handling.
- **FR-008**: Failover MUST preserve the request's output guarantees (including structured/validated output) regardless of which provider ultimately serves it.
- **FR-009**: Failover MUST be bounded and MUST NOT be triggered by deterministic failures — invalid input, other 4xx client errors, or a provider's content-policy/safety refusal. These are handled on the selected provider (degrade gracefully) rather than routed to the other provider, so failover never bypasses a safety block.
- **FR-010**: Every AI call MUST record which provider and model served it, and MUST indicate when a failover occurred (attempted vs serving provider).
- **FR-011**: Per-organisation spend caps and cost tracking MUST account for combined spend across both providers against the same organisation limits.
- **FR-012**: Changing an organisation's provider MUST take effect for subsequent requests without a redeploy.
- **FR-013**: Changing an organisation's provider MUST be restricted to authorised operator/admin roles, not arbitrary end users.
- **FR-014**: A misconfigured or unavailable provider for one organisation MUST NOT break AI for organisations using the other provider.
- **FR-015**: Embeddings and knowledge-base retrieval MUST remain on their current provider and are out of scope for provider switching.
- **FR-016**: An organisation MAY be marked "strict" (no cross-provider failover). For a strict organisation, an AI request MUST NOT be retried on the other provider; on its selected provider's outage the request MUST degrade gracefully (no user-facing 500) rather than crossing the provider boundary. Failover default is on; strict is the opt-out.
- **FR-017**: The system MUST bound how long it waits on an unresponsive provider, declaring it unavailable after a bounded per-attempt wait (target ~20 seconds), and MUST attempt at most one cross-provider failover per request — so a failed-over request does not hang and completes within roughly twice a normal call's latency.
- **FR-018**: The Microsoft AI Foundry connection MUST authenticate via Entra Managed Identity when Orbit runs on Azure, with no Foundry key or secret stored in code or environment variables (any required secret held in Key Vault). Key-based authentication is permitted only as a documented development / non-Azure fallback.

### Key Entities *(include if feature involves data)*

- **AI Provider**: A connected AI backend the platform can serve requests through (Anthropic, Microsoft AI Foundry). Holds its connection configuration and a mapping from task tier to a concrete model deployment. The Foundry connection authenticates via Entra Managed Identity (keyless) when running on Azure, with any secret held in Key Vault; key-based auth is a documented dev / non-Azure fallback only.
- **Organisation AI Provider Setting**: The per-organisation selection of the active provider, with a platform default (Anthropic), plus a "strict" flag that disables cross-provider failover for residency-bound organisations. Resolved per request.
- **Task-Tier Model Mapping**: Per-provider mapping from a task class (generation, classification, reasoning) to the concrete model that serves it, so the same feature works on either provider.
- **AI Usage Record**: The existing per-call usage/cost record, extended to attribute each call to the serving provider and model and to note failover.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: An operator can switch an organisation between the two providers and see it take effect on the organisation's next AI request, with no redeploy.
- **SC-002**: Every AI-backed feature produces a valid result on either provider. Because all features route through the three task tiers (generation, classification, reasoning), validation exercises all three tiers on both Anthropic and Foundry — confirming the shared path that serves 100% of the features.
- **SC-003**: When an organisation's selected provider is fully unavailable, at least 99% of its AI requests still succeed via automatic failover; only a both-providers-down condition causes failure.
- **SC-004**: Zero existing organisations change provider or AI behaviour when the feature ships (default preserved, no regressions).
- **SC-005**: 100% of AI usage records are attributable to the provider that served the request, enabling a per-provider cost and volume breakdown.
- **SC-006**: A single-provider outage produces no user-facing 500 across any AI feature.
- **SC-007**: When the active provider is unresponsive, the request completes via failover within approximately twice the latency of a normal single-provider call (bounded ~20s detection, one failover attempt) rather than hanging.

## Assumptions

- Provider selection is **per-organisation**, resolved per request from organisation settings, mirroring the established `industry_template` resolution pattern.
- Both providers' credentials/endpoints are configured **once at the application level**; organisations select among the connected providers rather than each supplying their own keys. (Per-tenant bring-your-own-key is out of scope for v1.)
- The Foundry connection uses **Entra Managed Identity** on Azure (keyless, secrets in Key Vault); on non-Azure hosts (e.g. today's Vercel) a key-based fallback is used and documented. Anthropic continues to use its existing API-key configuration.
- The **default** provider is Anthropic; all existing organisations are treated as Anthropic on launch.
- Foundry tier mapping uses **currently deployed Foundry models** — generation→Claude Sonnet, classification→GPT-4o (or Sonnet), reasoning→Claude Sonnet — with the exact mapping finalised during planning; parity models (Haiku/Opus equivalents) can refine it later without changing this spec.
- Failover applies to **transient/availability** failures, not to deterministic validation failures that would fail on any provider.
- Provider-switching control is **operator/admin-facing**; a self-serve, customer-facing provider picker is out of scope for v1 unless later requested.
- Cost-rate data for Foundry-served models will be maintained so combined cost tracking stays accurate; a missing rate degrades to "unknown cost," never a failed call.
- **Embeddings / knowledge-base retrieval (Voyage) are out of scope** and unchanged.
- The existing unified AI wrapper remains the single routing point for all AI calls; the one known direct-SDK call site is expected to be brought through the wrapper as part of this work.
