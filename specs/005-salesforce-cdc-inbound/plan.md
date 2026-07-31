# Implementation Plan: Real-Time Salesforce Inbound (CDC)

**Branch**: `005-salesforce-cdc-inbound` | **Date**: 2026-07-30 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/005-salesforce-cdc-inbound/spec.md`

## Summary

Replace Orbit's interim hourly inbound polling with near-real-time Salesforce ingestion, matching the proven .NET design in `SALESFORCE-INTEGRATION-ANALYSIS.md` §8/§9. The approach splits into two tiers: **one always-on "CDC shipper"** (a small Node/TypeScript service on Azure Container Apps) that owns the Salesforce Pub/Sub gRPC change stream per connected org, decodes Avro events, checkpoints a per-org replay cursor, and emits one Inngest event per changed record; and **Inngest + Supabase** for everything durable-but-event-driven — the per-record apply (ordered, idempotent), the resumable backfill (gap-recovery, manual, on-connect, and a one-time rollout retro-provision), and the on-connect org provisioning (External-Id field + CDC channel + FLS). The shipper never sees the token master key: it fetches per-org access tokens from an internal Orbit endpoint that reuses the existing `getValidToken` refresh path. Steady-state polling is retired behind a reversible feature flag.

## Technical Context

**Language/Version**: TypeScript on Node ≥22 (both Orbit app and the new shipper service).

**Primary Dependencies**: Next.js 15 (Orbit app + internal endpoints), Inngest ^4 (durable orchestration), Drizzle + Supabase Postgres (data), `@grpc/grpc-js` + `@grpc/proto-loader` + `avsc` (shipper: Salesforce Pub/Sub gRPC + Avro decode). Reuses existing `src/lib/crm/*` (mapping/routing/inclusion/coercion), `src/lib/salesforce.ts` (OAuth/token), and the outbound path.

**Storage**: Supabase Postgres. New: `salesforce_ingestion_state` (per-org replay cursor + subscription health), `salesforce_provisioning_status` (per-org readiness). Extends `contacts` (external-id already implied by `salesforce_id`), `crm_sync_receipts`/`checkpoints`, `salesforce_sync_logs` (reused), `organizations.salesforce_config` JSONB (sync mode flag).

**Testing**: Jest (unit, existing), Inngest function tests + golden tests (existing pattern in `src/inngest/__tests__/`), Playwright (e2e, existing). Shipper: Jest unit tests for decode/cursor/backoff (pure), plus an integration harness against a Salesforce developer org.

**Target Platform**: Orbit app on Azure App Service / Container Apps (existing target); the CDC shipper on **Azure Container Apps** — a single always-on replica (min=max=1).

**Project Type**: Web application (Next.js) + one companion long-running worker service (the shipper).

**Performance Goals**: Inbound change reflected in Orbit p95 < 60s (SC-001). One shipper instance sustains ≤50 connected orgs (SC-008).

**Constraints**: The Salesforce Pub/Sub CDC API is gRPC-only and requires a persistent bidirectional stream — it cannot run inside a serverless function (execution caps, no held sockets). Per-org API budget must be respected (backoff, no exhaustion). Consent/opt-out fields never written inbound (compliance).

**Scale/Scope**: <50 connected orgs at launch; single shipper instance multiplexing all streams over one shared gRPC channel; no sharding, no warm standby (v1). Lead + Contact objects only.

## Constitution Check

*GATE: evaluated against `Orbit on Azure` constitution v1.1.0.*

| Principle | Verdict | Notes |
|---|---|---|
| I. Azure-Native, Managed-First | ✅ PASS | Shipper runs on **Azure Container Apps** (PaaS), single replica — **no VM**. Orbit app stays on App Service/Container Apps. Every new runtime maps to a concrete managed Azure resource. |
| II. Everything as Infrastructure-as-Code | ✅ PASS | The Container App, its scaling (min=max=1), egress, identity, and secret references are defined in IaC (Terraform, aligning with the Azure landing-zone spec 004) and committed. No portal click-ops. |
| III. Secure by Default (NON-NEGOTIABLE) | ✅ PASS | Shipper authenticates to Orbit's internal token endpoint via **Managed Identity (Entra)** — no keys in shipper code; the token **master key stays solely in the Orbit app**; Salesforce tokens remain encrypted at rest. Required public egress is outbound-only to `api.pubsub.salesforce.com:7443`; the shipper exposes only a health endpoint (no inbound business traffic). Secrets via Key Vault references. |
| IV. Reversible, Low-Downtime Migration | ✅ PASS | Polling retirement is behind a per-environment `SF_INBOUND_MODE = cdc \| poll` flag — flipping back to `poll` restores the old path with no data change. The replay cursor + gap-recovery backfill guarantee no data loss across the cutover. |
| V. Simplicity & Ship-Date Discipline (YAGNI) | ✅ PASS | Single instance, ≤50 orgs, one shared gRPC channel, no sharding/HA/leader-election in v1 (see Clarifications). Reuses Orbit's entire existing CRM/OAuth/outbound layer rather than rebuilding. The one added deployable is justified below. |

**Result: PASS.** One justified deviation from "pure serverless" is tracked in Complexity Tracking (the always-on shipper).

## Project Structure

### Documentation (this feature)

```text
specs/005-salesforce-cdc-inbound/
├── plan.md              # This file
├── spec.md              # Feature spec (+ Clarifications)
├── research.md          # Phase 0 — decisions & rationale
├── data-model.md        # Phase 1 — entities, tables, transitions
├── quickstart.md        # Phase 1 — end-to-end validation guide
├── contracts/           # Phase 1 — internal contracts
│   ├── inngest-events.md         # event names + payload schemas
│   ├── internal-token-endpoint.md# shipper ↔ Orbit token contract
│   ├── cdc-record-event.md       # normalized change-event shape
│   └── provisioning.md           # org provisioning + status contract
└── tasks.md             # Phase 2 — created by /speckit-tasks (not here)
```

### Source Code (repository: `repostories/orbit`)

```text
repostories/orbit/
├── services/
│   └── salesforce-cdc-shipper/          # NEW always-on worker (Container Apps)
│       ├── src/
│       │   ├── main.ts                  # host: reconcile loop + health server
│       │   ├── subscription/            # per-org gRPC stream runner, supervisor, backoff
│       │   ├── pubsub/                   # proto, avro decode, replay-id hex, flow control
│       │   ├── cursor/                   # replay cursor read/commit (Supabase)
│       │   ├── discovery/               # which orgs are connected (Supabase)
│       │   ├── token/                    # internal-token-endpoint client
│       │   └── emit/                     # inngest event send per record
│       ├── proto/pubsub_api.proto       # vendored Salesforce Pub/Sub proto
│       ├── Dockerfile
│       └── package.json
├── src/
│   ├── inngest/
│   │   ├── salesforce-cdc-apply.ts      # NEW per-record apply (ordered, idempotent)
│   │   ├── salesforce-backfill.ts       # NEW resumable backfill (gap/manual/connect/retro)
│   │   ├── salesforce-provision.ts      # NEW on-connect org provisioning
│   │   ├── salesforce-keepalive.ts      # NEW dormant-token keep-alive cron
│   │   └── salesforce-sync.ts           # MODIFIED: retire inbound poll (flag-gated)
│   ├── lib/
│   │   ├── crm/**                        # REUSED (mapping/routing/inclusion/coercion)
│   │   ├── salesforce.ts                 # REUSED + extend (provisioning, describe helpers)
│   │   └── salesforce/
│   │       ├── cdc-decode.ts             # shared decode/replay helpers (used by shipper)
│   │       └── provisioning.ts           # NEW Tooling-API provisioning calls
│   └── app/api/
│       ├── internal/salesforce/ingestion-token/route.ts  # NEW shipper token endpoint
│       └── inngest/route.ts              # MODIFIED: register new functions
├── supabase/migrations/
│   ├── *_salesforce_ingestion_state.sql # NEW
│   └── *_salesforce_provisioning_status.sql # NEW
└── infra/                                # NEW Terraform: Container App + identity + secrets
```

**Structure Decision**: Web app + one worker. The persistent, non-serverless concern (the gRPC stream) is isolated in `services/salesforce-cdc-shipper/`; everything durable-but-event-driven lives in Orbit's existing Inngest app under `src/inngest/`. Shared pure logic (Avro decode, replay-id encoding, field mapping) lives in `src/lib/` so both the shipper and the Inngest functions import one implementation. This keeps the blast radius of the new deployable minimal and lets ~90% of the feature ride Orbit's existing stack.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| A second always-on deployable (CDC shipper) rather than pure serverless Inngest | Salesforce CDC is delivered **only** over a persistent bidirectional gRPC stream (Pub/Sub API). A serverless function cannot hold that socket open (execution caps, no long-lived connection), and there is no webhook/HTTP-push alternative for CDC. | Polling (the current approach) has up-to-1-hour latency and per-record state problems the spec exists to remove; bridging CDC via a broker just relocates the same persistent-connection requirement. The shipper is kept as small as possible (stream → decode → cursor → emit) with all business logic in Inngest. |
