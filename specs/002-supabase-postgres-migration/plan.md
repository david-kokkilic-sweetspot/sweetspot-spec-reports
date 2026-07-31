# Implementation Plan: Supabase → Postgres/Blob Application Migration

**Branch**: `002-supabase-postgres-migration` | **Date**: 2026-07-27 · **Re-scoped**: 2026-07-29 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/002-supabase-postgres-migration/spec.md`

## Summary

Move Orbit's **application layer** off the Supabase platform onto the target Postgres/Blob-shaped
stack, built and validated against the **001 local substrate** (Docker PostgreSQL 17 + Azurite):
**Drizzle ORM** replacing ~5,100 `supabase-js` call sites (schema replayed from the 283 migrations via
a compatibility shim), **Auth0** (`@auth0/nextjs-auth0` v4, dev tenant) replacing Supabase Auth, and
the **Azure Blob client** (`@azure/storage-blob`, on Azurite) replacing Supabase Storage. RLS stays
enforced **in the database**, repointed from `auth.uid()` to the validated Auth0 identity via
`SET LOCAL`/`current_setting`. Inngest is retained with only its DB connection repointed; Realtime is
dropped (zero usage). ~80% of effort is the mechanical, parallelisable call-site rewrite; the highest
risk is cross-tenant isolation during the security rebuild. **Real Azure provisioning + production
cutover are spec 004** — the same code 002 delivers runs unchanged when 004 lands it.

## Technical Context

**Language/Version**: Next.js 16 / React 19 / Node ≥ 22. Data access → Drizzle ORM. Auth → `@auth0/nextjs-auth0` v4 (`proxy.ts` on Next 16).

**Primary Dependencies**: Drizzle ORM + a Postgres driver (`postgres.js` / `node-postgres`); `@auth0/nextjs-auth0` v4; `@azure/storage-blob` (already added in 001); Inngest (retained). No Terraform/`azurerm`/`auth0` provider here — that is 004.

**Storage**: The **001 local substrate** — Docker PostgreSQL 17 (88 tables, 283-migration history is source of truth) + Azurite Blob (1 bucket, ~9 call sites). The identical Drizzle/RLS/Blob code targets Azure PG Flexible Server + Blob when 004 lands it.

**Testing**: Existing type-check + full test suite; **dedicated tenant-isolation suite** (the security gate); Auth0 E2E flows (sign-up/login/logout/refresh/MFA/reset) against the dev tenant; Inngest 47-function health check; CI static analysis for zero `supabase-js`/Realtime/storage references. All run against the substrate, offline.

**Target Platform**: The 001 local substrate (Docker pg17 + Azurite), macOS/Linux, offline. Azure is 004.

**Project Type**: Web application — platform re-architecture of the app layer (feature parity, no new product features).

**Constraints**: Zero cross-tenant exposure (NON-NEGOTIABLE, gated by the isolation suite). Schema unchanged (283-migration history is source of truth). Everything buildable offline on the substrate — no Azure account, no VPN. Auth0 used via a dev tenant locally.

**Scale/Scope**: ~5,100 query calls + 46 stored-procedure calls rewritten; 51 auth touchpoints; ~150 RLS policies; 8 user-referencing tables; 47 Inngest functions; ~9 storage call sites.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

The constitution (`Orbit on Azure`, v1.1.0) governs the **Azure hosting and migration end-state** —
now carried by the Azure-implementation spec (004). Like 001, feature 002 provisions **no Azure
resources** and has **no production surface**; it delivers app code validated on a local, disposable
substrate. Evaluation:

| Principle | Verdict | Notes |
| --- | --- | --- |
| I. Azure-Native, Managed-First | ✅ PASS (out of scope) | No Azure resources created here. The app targets the Azure-native *engines* (Postgres + Blob) via the local substrate; the Azure-native end-state is 004. **Auth0** (not Azure-native) is the one product choice — an outbound app dependency, justified in Complexity Tracking; the Azure-vs-Auth0 identity decision binds 004's tenant provisioning, not this local build. |
| II. Everything as IaC | ✅ PASS (phased) | 002 authors app code + the `db-replay` shim (committed, reproducible). The Azure/Auth0 Terraform is 004. |
| III. Secure by Default (NON-NEGOTIABLE) | ✅ PASS (phased) | Squarely inside III's phased-application clause — a pre-production build with dev-only `.env.local` values, no real secret, no customer data. The DB-enforced RLS (the one production-relevant security property) IS built and gated here by the isolation suite. Private networking / Managed Identity / Key Vault are 004. |
| IV. Reversible, Low-Downtime Migration | ✅ PASS (out of scope) | No Supabase→Azure cutover happens here; the substrate is wiped and rebuilt at will. The real cutover + rehearsed rollback are 004. |
| V. Simplicity & Ship-Date (YAGNI) | ✅ PASS | Schema unchanged; Realtime dropped not rebuilt; no new features; reuse 001's substrate + `db-replay`; pilot the rewrite before the full batch. |

**Gate result**: PASS. One tracked deviation (Auth0 vs Entra ID) — an app-layer product choice; the Azure security MUSTs bind 004, not this local build.

*Post-Phase-1 re-check*: design confirms the DB-enforced RLS session contract on the substrate and reuse of 001's engines/clients. **Still PASS.**

## Project Structure

### Documentation (this feature)

```text
specs/002-supabase-postgres-migration/
├── plan.md              # This file
├── research.md          # Phase 0 output (R1–R9, retargeted to the substrate; Azure/cutover → 004)
├── data-model.md        # Phase 1 output — app/data entities (no Azure topology; that is 004)
├── quickstart.md        # Phase 1 output — replay + validate on the substrate
├── contracts/
│   ├── rls-session.md       # DB-enforced tenant isolation (the critical contract)
│   ├── auth-user-mirror.md  # Auth0 ↔ local user mirror sync
│   └── storage.md           # Blob upload/download/delete (Azurite)
└── tasks.md             # Phase 2 output (/speckit-tasks)
```

### Source Code (target: Orbit application repo, `repostories/orbit/`)

```text
repostories/orbit/
├── docker-compose.local.yml   # EXISTING (001): pg17 + azurite substrate
├── scripts/local/
│   ├── initdb/00-extensions.sql   # EXISTING (001)
│   └── db-replay.sh               # NEW (002): compatibility shim + 283-migration replay ($DATABASE_URL)
├── db/
│   ├── schema.ts            # Drizzle schema (introspected from the replayed substrate DB)
│   ├── client.ts            # Drizzle client + per-request RLS session identity injection
│   └── migrations/          # RLS policy repoint (auth.uid() → current_setting('app.user_id'))
├── lib/
│   ├── auth/                # Auth0 integration (proxy.ts), login-sync to the user mirror
│   └── storage/             # @azure/storage-blob client (Azurite; replaces Supabase Storage)
├── src/**                   # ~5,100 call sites rewritten, module-by-module
├── inngest/                 # 47 functions, DB connection repointed to Drizzle/substrate
├── tests/isolation/         # tenant-isolation suite (security gate)
└── tests/auth/              # Auth0 E2E + mirror reconciliation
                             # NO infra/terraform/ here — that is spec 004
```

**Structure Decision**: 002 delivers app code + the `db-replay` shim into the Orbit repo, all
validated against the 001 substrate. The critical path is replay → Drizzle wiring → call-site rewrite;
auth (US3) + security (US2) run as a parallel branch with US3→US2 sequencing. The rewrite is
partitioned by module/table for parallel automation. No Azure resources, no Terraform — that is 004.

## Complexity Tracking

| Deviation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| **Auth0 instead of the constitution's Entra ID** (Principle I) | Spec constraint: identity must come from a vendor that signs SOC2/ISO 27001/SLA/DPA for enterprise procurement; Auth0 provides managed MFA/passkeys/social/reset; greenfield adoption (no passwords to migrate). Locally it is a dev tenant only. | Entra External ID (the Azure-native option) is deferred, not rejected — it was not confirmed to carry the required signed commitments, and switching identity providers mid-migration is costly. The Azure-vs-Auth0 production decision is finalised in 004 when the real tenant is provisioned. |
| **Adding a compatibility shim for the replay** (vs stripping Supabase objects) | The baseline `pg_dump`'s ~150 RLS policies reference `auth.uid()` (189×) and roles that do not exist on stock PG; you cannot strip a function embedded in policy bodies without dropping all RLS. A pre-seed shim lets the dump apply verbatim, preserving the RLS structure that US2 then repoints. | Sed-stripping 245 policies across 283 files is fragile and destroys the RLS structure US2 needs to repoint — more code, more risk, worse fidelity. |
