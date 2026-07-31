# Implementation Plan: Dual AI Providers (Anthropic + Microsoft AI Foundry)

**Branch**: `003-dual-ai-providers` | **Date**: 2026-07-27 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/003-dual-ai-providers/spec.md`

## Summary

Orbit routes every AI call through one wrapper — `generate()` in `src/lib/ai/client.ts` — which
today instantiates a single Anthropic client and resolves a model from a task class. This feature
introduces a **provider abstraction behind that same chokepoint** so each organisation can run on
either **Anthropic** or **Microsoft AI Foundry**, resolved per request from an org setting, with
**automatic cross-provider failover** (opt-out per org via a "strict" flag) and **per-provider
usage/cost attribution**. Both providers stay connected at once; the default is Anthropic so
existing orgs are unchanged.

Technical approach: extract the single model-call step inside `generate()` into an `AiProvider`
adapter interface (mirroring the proven `getAdapter(provider)` pattern in `src/lib/crm/`), with
`anthropic` and `foundry` adapters. `generate()` resolves the org's provider + failover order,
runs the existing attempt loop against the primary adapter, and on a failover-eligible error
(timeout / 5xx / connection / 429) retries the same request once against the other adapter —
unless the org is strict. Schema validation, spend caps, retries, and `ai_usage` logging stay
where they are; the usage row gains provider attribution. Foundry authenticates via Entra Managed
Identity on Azure (key-based fallback off-Azure).

## Technical Context

**Language/Version**: TypeScript 5.x, Next.js (App Router), Node 18+ runtime

**Primary Dependencies**: `@anthropic-ai/sdk` (existing, the Anthropic-direct connection); **new**:
`@azure/identity` (Managed Identity / `DefaultAzureCredential`) and `@azure-rest/ai-inference` (the
general Azure AI Foundry model-inference `ModelClient` — any deployed model through one connection;
resolved in [research.md](./research.md) D1). The two connections stay on separate clients.

**Storage**: Supabase Postgres (RLS). Touched tables: `organizations` (new provider columns),
`ai_usage` (provider attribution), `ai_model_cost_rates` / `MODEL_PRICING` fallback (rate rows for
Foundry model IDs). No new tables.

**Testing**: Jest (`npm test`, unit + integration, external SDKs mocked), `node:test`
(`npm run test:node`), Playwright E2E (post-merge). Per CLAUDE.md §13, provider adapters + failover
get explicit contract tests; `/verify` against a real/local backend before merge.

**Target Platform**: Vercel today (key-based Foundry fallback); Azure App Service / Container Apps
later (Managed Identity) — same code path, config-selected auth.

**Project Type**: Web application (Next.js full-stack in `repostories/orbit`).

**Performance Goals**: A failed-over request completes within ~2× a normal single-provider call;
active provider declared unavailable after ~20s per-attempt wait; at most one failover per request
(FR-017 / SC-007). No added latency on the non-failover happy path.

**Constraints**: No user-facing 500 on a single-provider outage (SC-006); 100% backward compatible
for existing orgs (default Anthropic, SC-004); every AI call attributable to its serving provider
(SC-005); all ~20 AI features work on either provider (SC-002).

**Scale/Scope**: ~20 AI-backed API routes + 3 Inngest jobs, all already funnelling through
`generate()`. One known direct-SDK bypass (`src/lib/activation/email-generation.ts`) is brought
through the wrapper as part of this work. Embeddings (Voyage) out of scope.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

The constitution (`Orbit on Azure`, v1.1.0) primarily governs the Hosting and Postgres-migration
specs. This feature introduces an Azure AI Foundry connection, so the security-relevant principles
apply to that surface. Evaluation:

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Azure-Native, Managed-First | ✅ PASS | Foundry is the Azure-native managed AI service; no self-managed compute introduced. The concrete resource is the existing AI Landing Zone Foundry endpoint. |
| II. Everything as Infrastructure-as-Code | ✅ PASS (scoped) | This feature is application code + a DB migration (tracked, forward-only per CLAUDE.md §6). The Foundry **resource** itself is provisioned by the hosting workstream's IaC, not hand-created here; the app consumes an endpoint + identity role assignment, which the hosting IaC owns. |
| III. Secure by Default (NON-NEGOTIABLE) | ✅ PASS with tracked exception | FR-018: Managed Identity on Azure, no Foundry key/secret in code/env, secrets in Key Vault. The **key-based dev/non-Azure fallback** is the constitution's permitted phased-application deferral (pre-production / non-Azure host) — see Complexity Tracking for the re-activation trigger. Managed Identity is the production posture and is not deferred there. |
| IV. Reversible, Low-Downtime Migration | ✅ PASS | Default stays Anthropic; the provider setting is per-org and reversible with no redeploy (FR-012). Rollback = flip the org back / disable the Foundry adapter; no data migration risk. |
| V. Simplicity & Ship-Date Discipline (YAGNI) | ✅ PASS | Reuses the single `generate()` chokepoint, the existing `getAdapter` pattern, and existing tables (`ai_usage.data_residency_region`, `ai_model_cost_rates`). New surface is two org columns + one adapter interface + one Foundry adapter. No per-tenant keys, no new tables, no provider registry beyond the two required. |

**Gate result: PASS.** One tracked exception (key-based auth fallback) recorded in Complexity Tracking.

**Post-design re-check (after Phase 1)**: Still PASS. The design (research D1–D7) reinforces it — Managed
Identity via `DefaultAzureCredential` passed to the inference `ModelClient` (III), reuse of the single `generate()`
chokepoint and existing `ai_usage` / `ai_model_cost_rates` tables (V), default-Anthropic reversibility
(IV), and Foundry as the managed Azure AI service (I). No new violation introduced; the exception set is
unchanged.

## Project Structure

### Documentation (this feature)

```text
specs/003-dual-ai-providers/
├── plan.md              # This file
├── research.md          # Phase 0 output — Foundry invocation + auth decisions
├── data-model.md        # Phase 1 output — entities, columns, migration
├── quickstart.md        # Phase 1 output — end-to-end validation guide
├── contracts/           # Phase 1 output — AiProvider interface + provider config contract
│   ├── ai-provider.contract.md
│   └── provider-selection.contract.md
├── checklists/
│   └── requirements.md  # Spec quality checklist (from /speckit-specify)
└── tasks.md             # /speckit-tasks output (NOT created here)
```

### Source Code (repository root: `repostories/orbit/`)

```text
repostories/orbit/
├── src/lib/ai/
│   ├── client.ts                  # MODIFY: generate() resolves provider + failover; call step delegated to adapter
│   ├── providers/                 # NEW: provider abstraction (mirrors src/lib/crm/)
│   │   ├── index.ts               #   getAiProvider(provider) registry + resolveOrgProvider(orgId)
│   │   ├── types.ts               #   AiProvider interface, AiProviderName, AiProviderError, ChatRequest/ChatResult
│   │   ├── anthropic.ts           #   AnthropicProvider — wraps existing anthropic.messages.create()
│   │   ├── foundry.ts             #   FoundryProvider — Azure AI Foundry via Managed Identity (key fallback)
│   │   └── model-map.ts           #   MODEL_BY_CLASS per provider (Anthropic map moved here; Foundry map added)
│   └── usage/log.ts               # MODIFY: accept + persist provider attribution on ai_usage
├── src/lib/activation/
│   └── email-generation.ts        # MODIFY: route through generate() instead of direct SDK (removes the one bypass)
├── src/app/api/admin/
│   └── ai-provider/route.ts       # NEW (or extend admin surface): operator sets org provider + strict flag
├── supabase/migrations/
│   └── <ts>_org_ai_provider.sql   # NEW: organizations.ai_provider + ai_provider_strict (+ CHECK, RLS unchanged)
└── src/lib/ai/
    └── __tests__/                 # NEW: adapter contract tests + failover unit tests (mock both SDKs)
```

**Structure Decision**: Single Next.js web app. The provider abstraction lives in a new
`src/lib/ai/providers/` folder that deliberately mirrors `src/lib/crm/` (registry `index.ts` +
`types.ts` contract + one file per adapter), so the pattern is already familiar to the codebase and
CLAUDE.md §5's "adapter pattern for new integrations" is honoured. All existing AI call sites are
untouched — they keep calling `generate()`.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| Key-based Foundry auth as a fallback (constitution III prefers Managed Identity exclusively) | Orbit runs on Vercel today, where Azure Managed Identity is unavailable; local dev likewise has no MI. A key/service-principal secret (held in the platform secret store / Key Vault) is required to exercise Foundry before the Azure cutover. | Managed-Identity-only would block all pre-Azure development and testing of the Foundry path. **Re-activation trigger:** when Orbit runs on Azure App Service / Container Apps, Managed Identity becomes the sole auth and the key fallback is removed. Tracked against the hosting workstream cutover. |
| Two new columns on `organizations` (vs a separate settings table) | Provider selection is a per-org scalar resolved per request, exactly like `industry_template` which already lives on `organizations`. | A separate table adds a join on the AI hot path for two scalar values with no independent lifecycle. YAGNI. |
