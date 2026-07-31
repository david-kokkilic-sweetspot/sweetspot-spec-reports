---
description: "Task list for Dual AI Providers (Anthropic + Microsoft AI Foundry)"
---

# Tasks: Dual AI Providers (Anthropic + Microsoft AI Foundry)

**Input**: Design documents from `/specs/003-dual-ai-providers/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/

**Tests**: Included — CLAUDE.md §13 (SDLC) makes unit + integration tests mandatory for substantial
work, shipped in the same PR. The contract tests C1–C9 in `contracts/provider-selection.contract.md`
are the source for the test tasks below.

**Organization**: Grouped by user story. All paths are relative to the Orbit repo:
`repostories/orbit/`.

> **Remediation note (post-`/speckit-analyze`)**: T007/T021 harden content-filter classification for
> **Foundry** (analyze H1); T014 adds the misconfigured-provider isolation test (FR-014, M1);
> T004 tracks the Managed-Identity RBAC role assignment (M2); T002/T008 gate on the concrete Foundry
> endpoint + deployment id (M3). T032 also guards that embeddings stay off provider selection (C1 /
> FR-015); T013 adds the in-flight resolve-once test (C2).

## Format: `[ID] [P?] [Story?] Description`

- **[P]**: can run in parallel (different files, no dependency on an incomplete task)
- **[Story]**: US1 / US2 / US3 (Setup, Foundational, Polish have no story label)

## Path Conventions

Web app (Next.js). Provider abstraction lives in `src/lib/ai/providers/` (mirrors `src/lib/crm/`).
Migrations in `supabase/migrations/`. Tests co-located under `src/lib/ai/providers/__tests__/`.

---

## Phase 1: Setup (Shared Infrastructure & Preconditions)

**Purpose**: Dependencies, config, module scaffold, and the cross-workstream preconditions the Foundry
path needs.

- [ ] T001 Add `@azure-rest/ai-inference` and `@azure/identity` to `repostories/orbit/package.json` (npm install; commit the updated `package-lock.json`)
- [ ] T002 [P] Document Foundry config in `repostories/orbit/.env.example`: `AZURE_FOUNDRY_BASE_URL` (Foundry **model-inference endpoint**, e.g. `https://<resource>.services.ai.azure.com/models`), `AZURE_FOUNDRY_API_KEY` (dev/non-Azure fallback only), and a note that Azure uses Managed Identity (scope `https://cognitiveservices.azure.com/.default`) + the Cognitive Services User RBAC role — with Vercel/Key Vault setup per CLAUDE.md §7. **Obtain and record the concrete endpoint and the Foundry deployment id (multimodal-capable — Claude Sonnet or GPT-4o — so content-analysis features work) here; preconditions for T008/T010/T015 (analyze M3).**
- [ ] T003 [P] Scaffold `repostories/orbit/src/lib/ai/providers/` with placeholder `index.ts` and `types.ts` (empty exports) so later tasks add to a real module path
- [ ] T004 [P] **Precondition (cross-workstream, analyze M2)**: with the hosting/IaC workstream, provision/confirm the **Cognitive Services User / Foundry User** RBAC role assignment for the app's Managed Identity on the Foundry resource. Required for the Azure Managed-Identity path (T010); without it, on-network calls still 403 (known Foundry env gotcha). The dev key-fallback path does not need this — note it as a blocker only for the Azure cutover, not for local/dev validation.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The provider abstraction substrate + schema. Every user story depends on this.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T005 Write forward-only, idempotent migration `repostories/orbit/supabase/migrations/<ts>_org_ai_provider.sql` — `organizations.ai_provider` (text NOT NULL DEFAULT `'anthropic'`, CHECK in (`'anthropic'`,`'azure_foundry'`)), `organizations.ai_provider_strict` (boolean NOT NULL DEFAULT false), `ai_usage.provider` (text NULL); all `ADD COLUMN IF NOT EXISTS`; no RLS change (per data-model.md §1–2)
- [ ] T006 [P] Define provider contract types in `repostories/orbit/src/lib/ai/providers/types.ts`: `AiProviderName`, `ChatRequest`, `ChatResult`, `AiProvider` interface, `AiProviderError` — reuse `Message`/`MessageContent`/`ImageBlock` from `src/lib/ai/client.ts` (per contracts/ai-provider.contract.md)
- [ ] T007 [P] Implement the shared failover classifier in `repostories/orbit/src/lib/ai/providers/classify.ts` — maps a thrown SDK error to `failoverEligible` (timeout/no-status/5xx/429 → true; 400 `content_filter`/other 4xx/401/403 → false), per research D5. **Handle both connections' safety-block forms (analyze H1): Foundry's OpenAI-style `400 content_filter` and `200` + `finish_reason: content_filter`, AND Anthropic-direct's Anthropic-style safety error / `stop_reason` — classify all of them as terminal / not failover-eligible, so a block is never routed around by failing over.**
- [ ] T008 [P] Define `MODEL_BY_PROVIDER` in `repostories/orbit/src/lib/ai/providers/model-map.ts` — move today's `MODEL_BY_CLASS` (Anthropic) here; Foundry map points generation/classification/reasoning at the configured Foundry deployment id **from T002** — per-tier model is pure config, swappable without touching `foundry.ts`/`generate()` (research D3)
- [ ] T009 [P] Implement `AnthropicProvider` in `repostories/orbit/src/lib/ai/providers/anthropic.ts` — wraps the existing module-level `anthropic` client; `chat()` calls `messages.create`, returns normalized `ChatResult` (incl. `stopReason`); maps errors via the classifier to `AiProviderError`; `isConfigured()` checks `ANTHROPIC_API_KEY`
- [ ] T010 [P] Implement `FoundryProvider` in `repostories/orbit/src/lib/ai/providers/foundry.ts` — construct `@azure-rest/ai-inference` `ModelClient` once against `AZURE_FOUNDRY_BASE_URL`: `new AzureKeyCredential(AZURE_FOUNDRY_API_KEY)` when the key is set, else `new DefaultAzureCredential()` (scope `https://cognitiveservices.azure.com/.default`) — research D1/D2. `chat()` **translates** the internal `ChatRequest` (Anthropic-shape system/messages/image blocks) into the inference chat-completions body and maps the response (`choices[0].message.content`, `usage.prompt_tokens/completion_tokens`, `finish_reason`) back to the normalised `ChatResult`; classifier surfaces the OpenAI-style `content_filter` (400 or `finish_reason`) so T007 catches a safety block. `isConfigured()` = endpoint set AND (key present OR a managed identity is available). Depends on T006, T007
- [ ] T011 Implement `getAiProvider(name)` registry + `resolveOrgProvider(orgId)` in `repostories/orbit/src/lib/ai/providers/index.ts` — registry mirrors `src/lib/crm/index.ts` (exhaustive switch); `resolveOrgProvider` reads the two `organizations` columns, cached in-process ~60s (same shape as `getPlanAiCaps`), fail-open to `{ provider:'anthropic', strict:false }` on read error (depends on T009, T010)

**Checkpoint**: Adapters, registry, resolution, and schema exist — user stories can begin.

---

## Phase 3: User Story 1 - An organisation runs its AI on its selected provider (Priority: P1) 🎯 MVP

**Goal**: Each org runs every AI feature through its selected provider (Anthropic or Foundry),
Foundry serving all tiers via Sonnet; default stays Anthropic so existing orgs are unchanged.

**Independent Test**: Set one org to Foundry and one to Anthropic, exercise the same features in each,
confirm each is served by its selected provider (via `ai_usage.provider`), and a third untouched org
still runs on Anthropic.

### Tests for User Story 1

- [ ] T012 [P] [US1] Adapter + routing tests in `repostories/orbit/src/lib/ai/providers/__tests__/adapters.test.ts` — invariants 1–4 + C1 (anthropic served) + C2 (foundry served, all tiers), both SDKs mocked
- [ ] T013 [P] [US1] Resolution + admin-guard tests in `repostories/orbit/src/lib/ai/providers/__tests__/resolution.test.ts` — C8 (resolveOrgProvider DB error → anthropic fallback) + C9 (admin route rejects non-admin with 403). **Plus (analyze C2): a provider change mid-request does not affect an in-flight call — `generate()` resolves the provider once at request start (≤60s cache), so an in-flight request completes on its original provider and only the next request sees the new value (spec in-flight edge case).**
- [ ] T014 [P] [US1] **Provider-isolation test (analyze M1 / FR-014)** in `repostories/orbit/src/lib/ai/providers/__tests__/misconfigured-provider.test.ts` — an org whose selected provider has missing/invalid credentials (`isConfigured()` false, or the adapter throws an auth error) degrades gracefully for that org, while a second org on the working provider is unaffected in the same run (no shared-state contamination)

### Implementation for User Story 1

- [ ] T015 [US1] Refactor `generate()` in `repostories/orbit/src/lib/ai/client.ts` to call `resolveOrgProvider(orgId)`, resolve `MODEL_BY_PROVIDER[provider][modelClass]`, and route the model-call step through `getAiProvider(provider).chat(...)` — single-provider path (no failover yet); keep `checkSpendCap`, schema-validation/corrective-retry, transient-retry, and logging exactly as today. A provider that is not `isConfigured()` for the org degrades gracefully (no user-facing 500), not affecting other orgs (FR-014)
- [ ] T016 [US1] Add `provider?: AiProviderName` to `AiUsageLogInput` and write `ai_usage.provider` in `repostories/orbit/src/lib/ai/usage/log.ts`; have `generate()` pass the serving provider (per data-model.md §2)
- [ ] T017 [US1] Add operator route `repostories/orbit/src/app/api/admin/ai-provider/route.ts` (PATCH) — reuse the admin guard used by `/api/admin/ai-cost-rates`; Zod body `{ orgId, provider, strict? }`; update the `organizations` columns; effect propagates on the org's next request (≤60s cache TTL — no redeploy, FR-012/FR-013)
- [ ] T018 [US1] Route the one direct-SDK bypass through the wrapper: `repostories/orbit/src/lib/activation/email-generation.ts` — replace the direct `anthropic.messages.create` with `generate()` so it honours the org's provider (FR-004 completeness)
- [ ] T019 [US1] Validate US1: `npm run typecheck && npm run lint`, run T012–T014; `/verify` quickstart Scenarios 1–2 against a real/local backend (anthropic org unchanged; foundry org serves all three tiers)

**Checkpoint**: MVP — an org can be switched to Foundry and runs every AI feature there; existing orgs unchanged.

---

## Phase 4: User Story 2 - AI features keep working when a provider has an outage (Priority: P2)

**Goal**: When the selected provider fails on an availability/capacity error, the same request is
retried once on the other provider; strict orgs never cross the boundary; deterministic failures and
content-filter blocks never fail over; only both-down fails (gracefully).

**Independent Test**: Force the selected provider to fail transiently → request completes via the
other provider (`metadata.failed_over=true`); mark the org strict → no failover; both down → graceful
failure with no user-facing 500.

### Tests for User Story 2

- [ ] T020 [P] [US2] Failover tests in `repostories/orbit/src/lib/ai/providers/__tests__/failover.test.ts` — C3 (transient/5xx/429/timeout → served by other provider), C6 (both fail → `AiGenerationError`, graceful, failure logged), C7 (schema validation still enforced on the failover provider's output)
- [ ] T021 [P] [US2] No-failover tests in `repostories/orbit/src/lib/ai/providers/__tests__/no-failover.test.ts` — C4 (400 `content_filter` → no failover), C5 (strict org outage → no failover, graceful). **Plus (analyze H1): a Foundry safety block as `400 content_filter` AND as `200` + `finish_reason: content_filter`, and an Anthropic-direct safety `stop_reason`, are all classified terminal → no failover (the block is never bypassed).**

### Implementation for User Story 2

- [ ] T022 [US2] Add the failover loop to `generate()` in `repostories/orbit/src/lib/ai/client.ts` — build `order = strict ? [selected] : [selected, other]`; on `AiProviderError` with `failoverEligible` and a remaining provider, retry the same request once on the other; else terminal → existing graceful-failure path (FR-006/FR-007/FR-009/FR-016)
- [ ] T023 [US2] Enforce bounds in `generate()` — a per-attempt request timeout (~20s target) so an unresponsive provider is declared unavailable, and at most one cross-provider failover per request (FR-017/SC-007)
- [ ] T024 [US2] Record failover on the usage row — set `metadata.failed_over` + `metadata.attempted_provider` when the serving provider differs from the selected one, in the `generate()` → `logAiUsage` call (FR-010)
- [ ] T025 [US2] Validate US2: run T020–T021; `/verify` quickstart Scenarios 3–6 (failover, strict, content-filter, both-down)

**Checkpoint**: US1 + US2 — provider selection with resilient failover and strict opt-out.

---

## Phase 5: User Story 3 - Usage and cost stay governable across both providers (Priority: P3)

**Goal**: Every call attributable to its serving provider; cost tracked per model across both
providers against the same org caps; operators see a per-provider breakdown.

**Independent Test**: Run AI on both providers for one org; confirm each `ai_usage` row is attributed
to the serving provider with a `cost_usd`, and a low org spend cap counts combined spend across both.

### Tests for User Story 3

- [ ] T026 [P] [US3] Attribution + cost tests in `repostories/orbit/src/lib/ai/providers/__tests__/usage-attribution.test.ts` — per-provider grouping, unknown-model → `cost_usd=0` + warning (never a failed call), and combined spend-cap trip across both providers

### Implementation for User Story 3

- [ ] T027 [US3] Ensure cost rates exist for Foundry model ids — add rows via the existing finance path / a seed migration under `repostories/orbit/supabase/migrations/`, and/or extend the `MODEL_PRICING` fallback in `repostories/orbit/src/lib/ai/usage/model-pricing.ts` (FR-011; only if the configured Foundry deployment id differs from the Anthropic-direct model id or is priced differently)
- [ ] T028 [US3] Set `ai_usage.data_residency_region` per provider in `repostories/orbit/src/lib/ai/usage/log.ts` (`'anthropic-direct'` vs `'azure-foundry-<region>'`), passed from `generate()`
- [ ] T029 [US3] Add the per-provider usage/cost breakdown for operators — extend the admin AI-usage view (near `src/app/admin/ai-cost-rates/`) or add a grouped query over `ai_usage.provider` (SC-005)
- [ ] T030 [US3] Validate US3: run T026; `/verify` quickstart Scenario 7 (per-provider attribution + combined spend cap)

**Checkpoint**: All three stories independently functional.

---

## Phase 6: Polish & Cross-Cutting Concerns

- [ ] T031 [P] Update `repostories/orbit/CLAUDE.md` AI section — `generate()` is now provider-abstracted; document per-org provider selection, failover + strict, and Foundry Managed-Identity/key auth
- [ ] T032 [P] Add a guard test asserting (a) no `anthropic.messages.create` / `@azure-rest/ai-inference` `ModelClient` call exists outside `src/lib/ai/providers/` (single-chokepoint invariant, mirrors CLAUDE.md §5's red-flag rule), **and (b) the embeddings / KB-retrieval (Voyage) path does not import or route through `generate()` / provider selection — proving embeddings stay on their current provider regardless of the org's LLM provider (analyze C1 / FR-015).**
- [ ] T033 Final validation before PR — run the full CLAUDE.md §13 gate set (`lint`, `format:check`, `typecheck`, `npm test -- --ci`, `npm run test:node`), the complete `quickstart.md`, `/code-review`, and `/security-review` (third-party credentials/identity touched)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no code dependencies. T004 (RBAC) is a cross-workstream precondition for the *Azure* MI path only — it does not block dev/key-path work.
- **Foundational (Phase 2)**: depends on Setup — **blocks all user stories**. T008/T010 depend on the concrete endpoint + deployment id captured in T002 (M3).
- **User Stories (Phase 3–5)**: all depend on Foundational. US1 is the MVP. US2 depends on US1's
  `generate()` refactor (T015) since it extends the same loop. US3 depends on US1's attribution write
  (T016). US2 and US3 do not depend on each other.
- **Polish (Phase 6)**: after the desired stories are complete.

### User Story Dependencies

- **US1 (P1)**: after Foundational. No dependency on US2/US3. Independently testable.
- **US2 (P2)**: extends `generate()` from US1 (T015/T022 touch the same file — serialise). Independently testable once wired.
- **US3 (P3)**: extends the usage-log path from US1 (T016/T028 same file — serialise). Independently testable.

### Within Each Story

- Tests written to fail first, then implementation (CLAUDE.md §13).
- Foundational adapters/registry before `generate()` wiring.
- `generate()` single-provider (US1) before failover (US2).

### Parallel Opportunities

- Setup: T002, T003, T004 in parallel (distinct concerns/files).
- Foundational: T006, T007, T008, T009 in parallel (distinct files); T010 after T006/T007; T011 after T009/T010; T005 (migration) parallel to all of them.
- US1 tests T012, T013, T014 in parallel. US2 tests T020, T021 in parallel.
- Cross-story: once Foundational is done, US1 must land first (owns the `generate()` refactor); then US2 and US3 can proceed in parallel by different developers, coordinating on the two shared files (`client.ts`, `usage/log.ts`).

---

## Parallel Example: Foundational Phase

```bash
# After T005 migration is drafted, build the adapter substrate in parallel:
Task: "Define provider contract types in src/lib/ai/providers/types.ts"        # T006
Task: "Implement failover classifier in src/lib/ai/providers/classify.ts"      # T007
Task: "Define MODEL_BY_PROVIDER in src/lib/ai/providers/model-map.ts"          # T008
Task: "Implement AnthropicProvider in src/lib/ai/providers/anthropic.ts"       # T009
# then, after T006/T007:
Task: "Implement FoundryProvider in src/lib/ai/providers/foundry.ts"           # T010
```

## Parallel Example: User Story 1 tests

```bash
Task: "Adapter + routing tests in src/lib/ai/providers/__tests__/adapters.test.ts"           # T012
Task: "Resolution + admin-guard tests in src/lib/ai/providers/__tests__/resolution.test.ts"  # T013
Task: "Provider-isolation test in src/lib/ai/providers/__tests__/misconfigured-provider.test.ts"  # T014
```

---

## Implementation Strategy

### MVP First (User Story 1 only)

1. Phase 1 Setup → 2. Phase 2 Foundational (blocks everything) → 3. Phase 3 US1 → **STOP & VALIDATE**
   (`/verify` an org switched to Foundry runs all features; existing orgs unchanged) → demo.

### Incremental Delivery

1. Setup + Foundational → substrate ready.
2. US1 → per-org provider selection working end-to-end (MVP). Deploy/demo.
3. US2 → automatic failover + strict. Deploy/demo.
4. US3 → per-provider cost governance. Deploy/demo.

Each story ships in its own small PR (CLAUDE.md §13.5 "small, isolated batches"), with its tests,
`/verify`, and an independent `/code-review` before merge.

### Parallel Team Strategy

Foundational is a shared push. US1 lands first (owns the `generate()` refactor). Then US2 and US3 can
be split across two developers — they touch different concerns, coordinating only on `client.ts` and
`usage/log.ts`.

---

## Notes

- [P] = different files, no incomplete-task dependency.
- `client.ts` is touched by T015 (US1) and T022/T023 (US2) — keep these serial; do not parallelise across those two stories on that file.
- `usage/log.ts` is touched by T016 (US1) and T028 (US3) — likewise serial.
- The Azure Managed-Identity path additionally depends on T004 (RBAC role assignment) landing; the dev key-path does not.
- Every AI call still flows through `generate()`; no call site outside `src/lib/ai/` changes.
- Commit after each task or logical group; verify tests fail before implementing.
