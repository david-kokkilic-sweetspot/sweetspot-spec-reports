# Implementation Plan: Local Development Environment (Azurite + Docker Postgres)

**Branch**: `001-local-dev-environment` | **Date**: 2026-07-28 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-local-dev-environment/spec.md`

## Summary

Stand up a local, cloud-free substrate that mirrors Orbit's **target** Azure-native data/storage
stack — **PostgreSQL 17** (standing in for Azure Database for PostgreSQL Flexible Server) and
**Azurite** (standing in for Azure Blob Storage) — via a single `docker-compose.local.yml`, so any
developer can reproduce it on their own laptop with no Azure, no Supabase, and no VPN. Scope is the
**empty engines only**: bring-up, the three required extensions, a dev connection contract, and a
connectivity smoke through the target clients. **The schema is not loaded here** — the 283-migration
replay and its Supabase-internal adaptation are inseparable from the data-access rewrite and are
delivered as spec 002, built against this substrate (per Clarifications). Real Azure/Terraform is a
separate team's Azure-implementation spec (004).

Technical approach: two containers (`pgvector/pgvector:pg17` + `azurite`) wired by compose with named
volumes and healthchecks; an init step that installs `vector`/`pg_trgm`/`pgcrypto` on first boot; a
committed `.env.local.example` connection contract; and a `tsx` smoke using the same clients the Azure
target uses (`pg`, `@azure/storage-blob`). Default host ports 5433 / 10000, both configurable.

## Technical Context

**Language/Version**: Tooling in TypeScript run via `tsx` (already a dev dependency); Docker Compose
for bring-up. The emulated engine is **PostgreSQL 17** (source `supabase/config.toml major_version = 17`).

**Primary Dependencies**: Docker + Docker Compose; images `pgvector/pgvector:pg17` and
`mcr.microsoft.com/azure-storage/azurite`. New dev-only npm deps for the smoke: `pg`,
`@azure/storage-blob`, `dotenv`. (App-side `pg`/Blob usage and the schema replay arrive with spec 002.)

**Storage**: Local PostgreSQL 17 (named volume `orbit_pgdata`), empty with the three extensions
installed; Azurite Blob (named volume `orbit_azuritedata`). No schema and no tables are created here —
the production schema is loaded by the 002 replay against this substrate.

**Testing**: A single `tsx` smoke (`scripts/local/smoke.ts`) exercising `pg` (connect + extensions +
trivial query) and `@azure/storage-blob` (container CRUD); Jest (`npm test`, present) remains for any
unit tooling.

**Target Platform**: Developer laptop, macOS/Linux, Docker + Node only. Offline-capable.

**Project Type**: Local developer infrastructure / environment-as-code inside the `repostories/orbit`
web app repo.

**Performance Goals**: One-command bring-up from clean in under 10 minutes (SC-001); fresh-clone
onboarding under 30 minutes (SC-004).

**Constraints**: Fully offline (no Azure/Supabase reachability, SC-002); idempotent reset (SC-003);
mirrors the Azure target, not Supabase (FR-011); dev-only secrets in `.env.local` (no production
secrets); default PG host port 5433 to dodge the Homebrew-on-5432 clash.

**Scale/Scope**: Single-developer, single-tenant local substrate. Two containers, one compose file,
one connection contract, one init SQL, one smoke script, plus package.json script wiring. No schema
replay (spec 002).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

The constitution (`Orbit on Azure`, v1.1.0) governs the **Azure hosting and migration end-state** — now
carried by the migration spec (002) and the future Azure-implementation spec (004). This feature
provisions **no Azure resources** and has **no production surface**; it is a local, disposable dev
substrate. Evaluation:

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Azure-Native, Managed-First | ✅ PASS (out of scope) | No Azure resources are created here. The substrate *emulates* the Azure-native engines (PostgreSQL + Blob) locally; the Azure-native end-state is spec 004. |
| II. Everything as Infrastructure-as-Code | ✅ PASS | The whole environment is defined as code — `docker-compose.local.yml` + init SQL + smoke + `.env.local.example`, committed and reproducible from clean. |
| III. Secure by Default (NON-NEGOTIABLE) | ✅ PASS (phased) | A local dev substrate with **no real secret and no customer data** — squarely inside III's phased-application clause. Secrets are dev-only in `.env.local`. Production posture is owned by spec 004. |
| IV. Reversible, Low-Downtime Migration | ✅ PASS (out of scope) | No Supabase→Azure cutover or data migration happens here; the substrate is wiped and rebuilt at will. The real cutover is spec 002/004. |
| V. Simplicity & Ship-Date Discipline (YAGNI) | ✅ PASS | Two stock images, one compose file, one init SQL, one smoke, existing clients. No schema replay, no orchestration, no extra services. The simplest thing that stands the engines up. |

**Gate result: PASS.** No violations; no Complexity Tracking exceptions.

**Post-design re-check (after Phase 1)**: Still PASS. The design (research R1–R6) introduces only local
containers, dev-only config, and reuse of the Azure clients — no new violation, no new Azure surface.

## Project Structure

### Documentation (this feature)

```text
specs/001-local-dev-environment/
├── plan.md              # This file
├── research.md          # Phase 0 — image/port/smoke decisions (R1–R6)
├── data-model.md        # Phase 1 — substrate components + connection contract (no schema)
├── quickstart.md        # Phase 1 — one-command bring-up + smoke + reset validation
├── contracts/
│   └── local-env.contract.md   # Compose (C1), connection (C2), smoke (C4), reset
├── checklists/
│   └── requirements.md  # Spec quality checklist (from /speckit-specify)
└── tasks.md             # /speckit-tasks output
```

### Source Code (repository root: `repostories/orbit/`)

```text
repostories/orbit/
├── docker-compose.local.yml        # NEW: postgres 17 (pgvector) + azurite, volumes, healthchecks (C1)
├── .env.local.example              # NEW: DATABASE_URL, AZURE_STORAGE_CONNECTION_STRING, ports (C2)
├── scripts/local/                  # NEW
│   ├── initdb/00-extensions.sql    #   CREATE EXTENSION vector/pg_trgm/pgcrypto on first boot (C1)
│   └── smoke.ts                    #   pg connect+extensions+query + Azurite blob CRUD, via tsx (C4)
└── package.json                    # MODIFY: add local:up/down/smoke scripts + devDeps
                                    #         (pg, @azure/storage-blob, dotenv)
```

**Structure Decision**: Environment-as-code lives at the Orbit repo root (compose + env example) with
tooling under `scripts/local/`, mirroring how a Next.js app carries its local infra. No `src/` code
changes and no `supabase/migrations/` usage — the schema replay and the app's switch to these engines
are spec 002.

## Complexity Tracking

No constitution violations — this table is intentionally empty. The one notable choice (adding `pg`,
`@azure/storage-blob`, `dotenv` as dev dependencies before spec 002 adds them to the app) is not a
violation: they are dev-only tooling for the substrate's own smoke, and they are the same clients the
Azure target uses, so the smoke faithfully exercises the target interface (SC-002).
