---
description: "Task list for Local Development Environment (Azurite + Docker Postgres)"
---

# Tasks: Local Development Environment (Azurite + Docker Postgres)

**Input**: Design documents from `/specs/001-local-dev-environment/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/local-env.contract.md, quickstart.md

**Scope**: Engines only — bring up Docker PostgreSQL 17 (pgvector) + Azurite, wire a `.env.local`
contract, and prove reachability with a connectivity smoke through the target clients. **No schema
replay, no Supabase-internal adaptation, no seed, no app boot** — those are spec 002, built against
this substrate (per spec Clarifications).

**Tests**: Not a TDD suite — the feature's verification IS the smoke (`scripts/local/smoke.ts`),
authored as an implementation task in US2.

**Organization**: Grouped by user story. All paths are relative to the Orbit repo:
`repostories/orbit/`.

## Format: `[ID] [P?] [Story?] Description`

- **[P]**: can run in parallel (different files, no dependency on an incomplete task)
- **[Story]**: US1 / US2 / US3 (Setup, Foundational, Polish have no story label)

## Path Conventions

Environment-as-code lives at the Orbit repo root (`docker-compose.local.yml`, `.env.local.example`)
with tooling under `scripts/local/`. No `src/` application code changes and no `supabase/migrations/`
usage — the app's switch to these engines and the schema replay are spec 002.

---

## Phase 1: Setup (Shared Infrastructure & Preconditions)

**Purpose**: Dev tooling and the connection contract the substrate exposes.

- [X] T001 Add dev dependencies `pg`, `@azure/storage-blob`, `dotenv` AND the `local:*` npm scripts to `repostories/orbit/package.json` — `local:up` = `docker compose -f docker-compose.local.yml up -d --wait` (**`--wait` so it blocks until the healthchecks pass and the smoke can't race Postgres boot**), `local:down` = `docker compose -f docker-compose.local.yml down -v`, `local:smoke` = `tsx scripts/local/smoke.ts`; run `npm install` and commit the updated `package-lock.json` (plan Technical Context, research R5)
- [X] T002 [P] Create `repostories/orbit/.env.local.example` with the connection contract: `POSTGRES_PORT` (default 5433), `AZURITE_BLOB_PORT` (default 10000), `DATABASE_URL` (`postgresql://orbit:orbit@localhost:${POSTGRES_PORT}/orbit`), `AZURE_STORAGE_CONNECTION_STRING` (Azurite dev string at the local Blob endpoint), and an optional `ANTHROPIC_API_KEY` — dev-only values, no production secrets (contract C2, research R4/R6)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The compose stack + extension init. Every user story depends on these.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [X] T003 Author `repostories/orbit/docker-compose.local.yml` — service `postgres` (`pgvector/pgvector:pg17`, host `${POSTGRES_PORT:-5433}`→container 5432, `POSTGRES_USER/PASSWORD/DB=orbit`, named volume `orbit_pgdata`, `pg_isready` healthcheck, mounts `scripts/local/initdb/` into `/docker-entrypoint-initdb.d`) and service `azurite` (`mcr.microsoft.com/azure-storage/azurite`, Blob-only on host `${AZURITE_BLOB_PORT:-10000}`, named volume `orbit_azuritedata`); no external network (contract C1, research R1/R2/R6)
- [X] T004 [P] Create `repostories/orbit/scripts/local/initdb/00-extensions.sql` — `CREATE EXTENSION IF NOT EXISTS vector;` + `pg_trgm` + `pgcrypto`, so all three exist as soon as PostgreSQL 17 starts (contract C1, research R1)

**Checkpoint**: `local:up` brings both containers up healthy with the three extensions present.

---

## Phase 3: User Story 1 - Bring the local stack up with one command (Priority: P1) 🎯 MVP

**Goal**: One command stands up an empty PostgreSQL 17 (with the three extensions) + Azurite, offline.

**Independent Test**: On a clean machine with the images already pulled and no cloud access, run
`local:up`; confirm both containers healthy and `\dx` shows `vector`/`pg_trgm`/`pgcrypto`, with no
Azure/Supabase reachability at runtime.

- [X] T005 [US1] Validate US1: `cp .env.local.example .env.local`, then `npm run local:up` from clean → both containers report healthy (the `--wait` blocks until healthchecks pass); `\dx` shows `vector`/`pg_trgm`/`pgcrypto`; confirm no cloud backend (Azure/Supabase) was reached at runtime (quickstart US1, SC-001)

**Checkpoint**: MVP — the empty substrate exists with extensions, offline, from one command.

---

## Phase 4: User Story 2 - Prove the substrate is usable through the target clients (Priority: P2)

**Goal**: A live PostgreSQL connection (extensions + trivial query) and a full Blob CRUD cycle succeed
against Azurite — through the same clients the Azure target uses.

**Independent Test**: With the stack up, run `local:smoke`; confirm the three extensions present + a
trivial `SELECT` succeed, and Blob create/put/get/delete succeed against Azurite, all offline.

- [X] T006 [US2] Author `repostories/orbit/scripts/local/smoke.ts` (run via `tsx`) — with `pg`: connect using `DATABASE_URL` (a short connect-retry, e.g. a few 500ms attempts, as belt-and-suspenders in case it runs before `--wait` settles), assert `vector`/`pg_trgm`/`pgcrypto` present in `pg_extension`, run a trivial `SELECT 1`; with `@azure/storage-blob` against Azurite: create a container, put an object, get it back (bytes match), delete it; exit non-zero on any failure (contract C4, research R5)
- [X] T007 [US2] Validate US2: `npm run local:smoke` → extensions + trivial query pass and Blob create/put/get/delete all succeed against Azurite; re-run with host network disabled to confirm zero cloud dependency (SC-002; contract C4)

**Checkpoint**: US1 + US2 — the substrate is proven reachable at the engine/Blob level through the real clients.

---

## Phase 5: User Story 3 - Reset and reproduce a clean environment (Priority: P3)

**Goal**: Teardown and re-bring-up return an identical clean state; a fresh-clone teammate reaches a
green stack from the docs.

**Independent Test**: up→smoke green, write data, `local:down`, then repeat → identical state and smoke
green both times; separately, follow the quickstart from a fresh clone.

- [X] T008 [US3] Ensure `local:down` = `docker compose -f docker-compose.local.yml down -v` wipes both named volumes (`orbit_pgdata`, `orbit_azuritedata`) so the next bring-up starts from empty (reset contract, FR-006)
- [X] T009 [US3] Validate idempotency: run up→smoke (green), write a Blob object / create a table, `local:down`, then up→smoke again → the environment returns to the same known clean state and the smoke passes both times (SC-003)
- [X] T010 [P] [US3] Document the local environment — add a "Local development environment" section to `repostories/orbit/README.md` (or `docs/local-dev.md`) covering the one-command bring-up, reset, port overrides, and the quickstart flow, so any teammate reaches a green stack with no cloud account or VPN (FR-009, SC-004)

**Checkpoint**: All three stories independently functional.

---

## Phase 6: Polish & Cross-Cutting Concerns

- [X] T011 [P] Update `repostories/orbit/CLAUDE.md` — document the local Azure-parity substrate (Docker PostgreSQL 17 + Azurite), the `local:*` scripts, that it stands up *empty* engines, and that the schema replay + app switch are spec 002 (developed against this substrate)
- [X] T012 [P] Ensure `repostories/orbit/.gitignore` ignores `.env.local` (dev secrets never committed) and any local volume/artifact paths
- [X] T013 Final validation before PR — run the full `quickstart.md` end to end on a clean checkout (up → smoke → down -v → re-up idempotent), confirm it works offline, and confirm `.env.local.example` contains no production secrets

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no dependencies. T001 (package.json) and T002 (.env example) are independent.
- **Foundational (Phase 2)**: depends on Setup — **blocks all user stories**. T003 (compose) references the initdb mount T004 creates; author them together (T004 [P]).
- **User Stories (Phase 3–5)**: all depend on Foundational. US1 is the MVP. US2 needs the stack up (US1). US3 exercises the full up→smoke cycle across a reset.
- **Polish (Phase 6)**: after the desired stories are complete.

### User Story Dependencies

- **US1 (P1)**: after Foundational. Independently testable (containers up + extensions).
- **US2 (P2)**: after US1 (the smoke connects to the running stack).
- **US3 (P3)**: after US1+US2 (it exercises the full up→smoke cycle across a reset).

### Within Each User Story

- Extensions (T004) come up at container init, before the smoke connects.
- Author the smoke (T006) before validating it (T007).

### Parallel Opportunities

- Setup: T001 and T002 in parallel (package.json vs .env example).
- Foundational: T003 and T004 in parallel (compose vs initdb SQL).
- Polish: T010, T011, T012 all in parallel (README vs CLAUDE.md vs .gitignore).

---

## Parallel Example: Foundational

```bash
# Foundational, in parallel:
Task: "Author docker-compose.local.yml (postgres pgvector:pg17 + azurite)"   # T003
Task: "Create scripts/local/initdb/00-extensions.sql"                        # T004
```

---

## Implementation Strategy

### MVP First (User Story 1 only)

1. Phase 1 Setup → 2. Phase 2 Foundational (blocks everything) → 3. Phase 3 US1 → **STOP & VALIDATE**
   (`local:up` → healthy stack + extensions, offline) → demo the substrate.

### Incremental Delivery

1. Setup + Foundational → containers defined.
2. US1 → one-command bring-up (MVP). Demo.
3. US2 → the smoke proves engine/Blob reachability through the target clients. Demo.
4. US3 → idempotent reset + onboarding docs. Demo.

---

## Notes

- [P] = different files, no incomplete-task dependency.
- Tests: no TDD suite requested; the smoke (T006) is the verification deliverable. Schema replay and app boot-and-serve are spec 002.
- `package.json` is touched only by T001 (deps + scripts); everything else is new files, so parallelism is safe.
- Ports default to 5433 / 10000 and are overridable in `.env.local` (avoids the Homebrew-PG-on-5432 clash).
- Secrets stay dev-only in `.env.local`; `.env.local` is git-ignored (T012).
- No `supabase/migrations/` replay here — 001 delivers empty engines; the 283-migration replay + Supabase adaptation is spec 002, built against this substrate.
- **Commit cadence**: commit once at each phase **Checkpoint** (logical group), not per task. Message: `feat(001): <phase> — <summary>` (e.g. `feat(001): foundational — compose + extensions init`). Do **not** add a `Co-Authored-By: Claude` trailer. Expected ~6 commits: Setup → Foundational → US1 → US2 → US3 → Polish.
