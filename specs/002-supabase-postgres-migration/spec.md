# Feature Specification: Supabase → Postgres/Blob Application Migration

**Feature Branch**: `002-supabase-postgres-migration`

**Created**: 2026-07-27 · **Re-scoped**: 2026-07-29

**Status**: Draft

**Input**: Re-scope of feature 002. Originally written (2026-07-27) as a full Supabase→Azure
migration including Azure infrastructure and production cutover. Re-scoped to match the 001
rescope: **002 is now the application-layer migration off Supabase**, built and validated against
the **001 local substrate** (Docker PostgreSQL 17 + Azurite). Real Azure resources (Terraform, PG
Flexible Server, private networking, Managed Identity, Key Vault) and the production cutover move to
the **Azure-implementation spec (004)**, owned by a separate team. The same data-access, auth, RLS,
and storage code 002 delivers runs unchanged against Azure when 004 lands it — that is the whole
point of the substrate mirroring the Azure-native engines.

## Clarifications

### Session 2026-07-29

- Q: Does 002 stand up Azure and cut production over, or does it deliver the app migration against the local substrate? → A: App migration against the **001 local substrate**. 002's Definition of Done is: the app's data access runs on Drizzle against local PostgreSQL 17, auth runs on Auth0 (dev tenant) with the local user mirror, RLS is enforced in the DB and the isolation suite is green, storage runs on `@azure/storage-blob` against Azurite, Inngest is repointed, and zero `supabase-js` remains. The **data/storage/RLS plane is fully offline** on the substrate; the one network dependency is the **Auth0 dev tenant** for auth flows (identity is a hosted service used via a dev tenant, per 001). Provisioning real Azure and cutting production over is **spec 004**.
- Q: Where does the "strip Supabase-internal objects" replay actually happen and how? → A: In 002, replaying the 283 migrations onto the substrate. The baseline is a Supabase `pg_dump` whose RLS policies reference `auth.uid()` (189×, 63 files) and roles `anon`/`authenticated`/`service_role`; these do not exist on stock PostgreSQL. Replay uses a **compatibility pre-seed (shim)** — create the roles + an `auth` schema with stub `auth.uid()/role()/jwt()` + `extensions`/`storage`/`supabase_functions` schemas — so the dump applies verbatim, then 002 **repoints the ~150 policies** to `current_setting('app.user_id')` as part of the Auth0 cutover. This reuses 001's `db-replay` mechanism, parameterized by `DATABASE_URL`.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Application data served through Drizzle on the substrate (Priority: P1)

Every part of Orbit that reads or writes data does so against local PostgreSQL 17 through a new
Drizzle-based data-access layer, with no dependency on the proprietary Supabase client. The existing
schema (88 tables, 283-file migration history) is reproduced unchanged as the ongoing source of
truth. The same Drizzle code runs against Azure PostgreSQL when 004 lands it.

**Why this priority**: This is ~80% of the work and the core of the migration — until data access
runs on stock Postgres, nothing else can move. It is the MVP slice.

**Independent Test**: Point a build at the **local substrate** (`DATABASE_URL` → Docker pg17),
exercise the full test suite and a representative set of app flows, and confirm all reads/writes
succeed with zero references to the legacy Supabase client remaining.

**Acceptance Scenarios**:

1. **Given** an empty local PostgreSQL 17 (001 substrate), **When** the 283 migrations are replayed with the compatibility shim, **Then** all 88 tables and required extensions exist and the measured table/object counts reconcile against the 88-table structure the migration history defines (offline — no Supabase/cloud access needed).
2. **Given** the rewritten data access, **When** the app and background jobs run against the substrate, **Then** all ~5,100 query calls and 46 stored-procedure calls succeed and zero legacy-client imports remain.
3. **Given** the running app, **When** the type-check and full test suite run, **Then** they pass.

---

### User Story 2 - Multi-tenant isolation preserved in the database (Priority: P1)

No customer can ever read or write another customer's data. Isolation stays enforced at the database
layer (not in application code), driven by the validated Auth0 identity injected into the session, so
a single missed check cannot leak data across tenants. Proven on the local substrate; identical on
Azure PG in 004.

**Why this priority**: Cross-customer exposure is the single highest-severity risk in the project;
the migration cannot be trusted without this guarantee proven.

**Independent Test**: Run the dedicated tenant-isolation suite against local PostgreSQL, attempting
cross-tenant reads and writes; confirm zero succeed.

**Acceptance Scenarios**:

1. **Given** all ~150 isolation policies repointed to `current_setting('app.user_id')` and active, **When** the isolation suite runs on the substrate, **Then** it records zero cross-tenant reads or writes.
2. **Given** a validated Auth0 token, **When** a session is established, **Then** the tenant identity is set at the database session (`SET LOCAL`) and every policy resolves against it.
3. **Given** an invalid or missing token, **When** a query is attempted, **Then** no tenant data is returned.

---

### User Story 3 - Enterprise-grade authentication via Auth0 (Priority: P1)

Users sign in through Auth0 (a provider that can carry formal compliance commitments — SOC 2, ISO
27001, SLA, DPA). All login-related flows work against an Auth0 **development tenant**, and the local
user mirror keeps database user references valid. (Identity is a hosted service used via a dev tenant
locally — it is not emulated in the substrate; production tenant provisioning is 004.)

**Why this priority**: Identity is a hard replacement touching 51 points and 8 tables; isolation
(US2) depends on the identity it produces.

**Independent Test**: Run end-to-end auth tests (sign-up, login, logout, session refresh, MFA,
password reset) against the Auth0 dev tenant and confirm all pass, and that a login creates/updates
the local mirror record.

**Acceptance Scenarios**:

1. **Given** the Auth0 dev tenant, **When** sign-up, login, logout, session refresh, MFA and password reset are exercised, **Then** all flows pass end-to-end.
2. **Given** a successful login, **When** the session is established, **Then** the local user mirror is created or updated, keyed on the Auth0 subject identifier.
3. **Given** the 8 user-referencing tables, **When** user references are resolved, **Then** they resolve against the local mirror, not the external provider directly.

---

### User Story 4 - Files served through the Azure Blob client on Azurite (Priority: P2)

The single file bucket and its ~9 call sites move from the Supabase storage client to
`@azure/storage-blob`, exercised locally against **Azurite** (the same client the Azure Blob target
uses — 001 proved it works). Transferring the real bucket objects with checksum verification is a
data-movement step and belongs to the 004 cutover.

**Why this priority**: Required for feature parity but small and low-risk; does not block the
critical path.

**Independent Test**: Exercise upload, download and delete through the app against Azurite; confirm
each path succeeds and no legacy storage-client references remain.

**Acceptance Scenarios**:

1. **Given** the app on the substrate, **When** upload, download and delete are exercised against Azurite, **Then** each path succeeds through `@azure/storage-blob` and no legacy storage-client references remain.
2. **Given** the migrated storage layer, **When** the CI static check runs, **Then** zero Supabase storage-client references remain.

---

### User Story 5 - Background jobs repointed to the substrate (Priority: P2)

The Inngest background-job framework is retained; only its database connection is repointed to the
Drizzle/local client, so all 47 functions keep running against local PostgreSQL.

**Why this priority**: Parity requirement; low effort (repoint connection) but must be verified so
scheduled work does not silently break.

**Independent Test**: Run the 47 functions (scheduled and event-driven) against the substrate and
confirm success.

**Acceptance Scenarios**:

1. **Given** the repointed connection, **When** each of the 47 functions runs against the substrate, **Then** all execute successfully.
2. **Given** the repointed jobs, **When** scheduled and event-driven triggers fire, **Then** each reads/writes the local database correctly.

---

### Edge Cases

- **Migration replay is not clean on stock Postgres**: the baseline `pg_dump`'s RLS policies reference `auth.uid()` and roles `anon`/`authenticated`/`service_role` that do not exist on stock PostgreSQL. The replay pre-seeds a compatibility shim (roles + `auth`/`storage`/`extensions`/`supabase_functions` schemas + stub `auth.uid()/role()/jwt()`) so the dump applies verbatim; a failed replay blocks the critical path and must surface early.
- **Local user mirror drifts from Auth0**: sync must happen on every login (not on a schedule), with a reconciliation check, to avoid broken foreign keys across the 8 tables.
- **Auth0 dev-tenant outage or latency on the login path**: requires JWKS caching; a production-grade auth-unavailability incident response is a 004 concern.
- **Pooled-connection identity bleed**: a `SET LOCAL` identity must never survive connection release into another request — pinned to the transaction, asserted by the isolation suite.
- **`extensions.digest()` / `supabase_functions.http_request` references** in the migration history resolve only if the shim creates those schemas/functions (or the replay strips the dead ones).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001** (Must): Orbit's data access targets **stock PostgreSQL 17** — locally the 001 substrate (Docker pg17), with `pgvector`, `pg_trgm`, `pgcrypto` verified. The same Drizzle code runs unchanged against Azure Database for PostgreSQL Flexible Server when 004 lands it.
- **FR-002** (Must): The existing schema is reproduced from the 283 migration files — clean replay from empty onto the substrate via a compatibility shim (roles + `auth`/`storage`/`extensions`/`supabase_functions` schemas + stub `auth.uid()/role()/jwt()`), the full schema present with counts **measured on the substrate: 169 base tables + 2 views at head** — the "88" is the April-30 `initial_schema` pg_dump baseline (a subset); the 282 later migrations grow it to 169 — reconciled with no cloud access. NOTE (measured in T005): reaching a clean 283/283 replay required 11 migration-history fixes to `deop/integration` (a lost-merge restore, a duplicate delete, 2 ordering reorders, 6 content fixes), and the shim also needed `postgres` SUPERUSER + managed-role grants — so the schema did **not** replay with zero manual fix-ups as originally assumed. Reuses 001's `DATABASE_URL`-parameterized `db-replay` so 004 runs the identical replay (and inherits the same fixes) against Azure.
- **FR-003** (Must): All data access is migrated from `supabase-js` to Drizzle ORM — all ~5,100 query calls and 46 stored-procedure calls rewritten, zero `supabase-js` imports in app or job code, type-check and full test suite pass.
- **FR-004** (Must): Authentication is replaced by Auth0 (`@auth0/nextjs-auth0` v4) — all 51 auth touchpoints migrated; sign-up, login, logout, session refresh, MFA and password-reset pass end-to-end against the Auth0 **dev tenant**.
- **FR-005** (Must): A local user mirror in the database stays in sync with Auth0 — keyed on the Auth0 subject identifier, created/updated on every login; user references in all 8 tables resolve against the mirror.
- **FR-006** (Must): Row-Level Security remains enforced in the database, driven by the validated Auth0 identity set into the session (`SET LOCAL` / `current_setting('app.user_id')`) — all ~150 policies repointed and active on the substrate; the tenant-isolation suite passes with zero cross-tenant reads/writes.
- **FR-007** (Must): File storage moves to the Azure Blob client (`@azure/storage-blob`), exercised against Azurite — all ~9 call sites migrated, upload/download/delete verified, no legacy storage-client references remain. (Real object transfer + checksums = 004.)
- **FR-008** (Must): The Inngest framework is retained with only its database connection repointed to the Drizzle/local client — all 47 functions execute against the substrate; scheduled and event-driven triggers verified.
- **FR-009** (Must): Supabase Realtime is decommissioned with no replacement — audit confirms zero runtime dependency and no Realtime client code remains.
- **FR-010** (Should): Database triggers and functions are ported unchanged and behaviourally verified against current outputs on the substrate.

### Non-Functional Requirements

- **NFR-001**: No cross-customer data exposure (highest-severity; enforced by FR-006, gated by the isolation suite on the substrate before 002 is considered done).
- **NFR-002**: The 283-file migration history remains the source of truth for schema going forward.
- **NFR-003**: No `supabase-js` / Supabase Realtime / Supabase storage-client reference remains in the app + job code (verified by CI static analysis).
- **NFR-004**: The data-access rewrite is decomposed for parallel automation — piloted on one module first (to prove it is mechanical enough) before the full batched rewrite.

**Technology decisions (recorded, settled):** Data access → Drizzle ORM · Auth → Auth0 (dev tenant locally, production tenant in 004) · Security → RLS in the database, repointed to the app session · Files → `@azure/storage-blob` (Azurite locally, Azure Blob in 004) · Realtime → dropped · Background jobs → Inngest retained.

### Key Entities

- **Local user mirror**: database record of each user, keyed on the Auth0 subject identifier; the anchor for the 8 user-referencing tables.
- **Tenant/customer identity**: the validated Auth0 identity injected into the database session that all isolation policies resolve against.
- **Isolation policy set**: the ~150 Row-Level Security policies enforcing per-tenant access at the database layer.
- **Schema migration history**: the 283 migration files defining the 88-table schema; source of truth.
- **Compatibility shim**: the pre-seed (roles + auth/storage/extensions/supabase_functions schemas + stub functions) that lets the Supabase `pg_dump` baseline replay on stock PostgreSQL 17.
- **Background-job functions**: the 47 retained Inngest functions whose only change is the database connection.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: The migrated app boots and serves against the 001 local substrate — full test suite + a representative flow set green. The data/storage/RLS plane runs with zero cloud dependency (offline); auth E2E uses the Auth0 dev tenant (the sole network dependency).
- **SC-002**: Zero cross-tenant data-exposure, verified by the isolation suite on the substrate.
- **SC-003**: All authentication flows (sign-up, login, logout, session refresh, MFA, reset) pass end-to-end against the Auth0 dev tenant, and a login upserts the local user mirror.
- **SC-004** (revised per T005 measurement): 100% of the 283 migrations replay onto the empty substrate via the shim — **169 base tables + 2 views** at head (the "88" is the `initial_schema` baseline, a subset) + three extensions (`pgvector`/`pg_trgm` in `public`, `pgcrypto` in `extensions`). Reaching a clean replay required **11 migration-history fixes** to `deop/integration` (lost-merge restore, duplicate delete, 2 ordering reorders, 6 content fixes) plus a shim that grants `postgres` SUPERUSER + managed-role schema access — the migrations did **not** replay with zero manual fix-ups as originally assumed. The measured counts reconcile against the migration history; the identical fixes carry to the 004 Azure replay.
- **SC-005**: Zero references to the legacy Supabase data/storage/Realtime clients remain in shipped app and job code (verified by static analysis in CI).
- **SC-006**: All 47 background-job functions execute successfully against the substrate (scheduled + event-driven).

## Assumptions

- **A1**: The data-access rewrite is mechanical enough to execute with heavy parallel automation — piloted first (NFR-004) to prove it before committing the full batch.
- **A2**: "Inngest unchanged" means keep the framework and repoint its database calls, not freeze every job file.
- **A3**: The 283 migration files replay cleanly onto stock PostgreSQL 17 once the compatibility shim pre-seeds the Supabase-internal roles/schemas/functions.
- **A4**: Auth0 is used via a **development tenant** locally; no user base exists (greenfield), so authentication is a fresh build, not a credential migration (no passwords to import).
- **A5**: The substrate mirrors the Azure-native engines faithfully (001), so Drizzle/RLS/Blob code validated locally runs unchanged against Azure in 004 — tuning (pool sizing, cold connections) aside.

## Out of Scope (Deferred)

1. **Real Azure resources and the production cutover** — Terraform (VNet, PG Flexible Server, Storage Account, Key Vault, Managed Identity, private endpoints), the Auth0 **production** tenant provisioning, the Supabase→Azure data sync + row-count/checksum parity, the parallel-run cutover, rollback, Supabase decommission, and p95 baseline/re-measure. All owned by the **Azure-implementation spec (004)**, seeded by `_archive/001-azure-host`. 002 delivers code that 004 lands.
2. **Production security posture** (private networking, Managed Identity DB auth, Key Vault secret wiring) — 004; locally the substrate uses dev-only `.env.local` values.
3. Redesigning the database schema beyond what the migration mechanically requires.
4. Rebuilding Realtime functionality (zero usage — dropped, not replaced).
5. Adding new product features (feature parity is the bar).
6. Choosing the Azure compute host — the Azure-implementation spec (004).

## Open Decisions

| Decision | Owner | Needed by | Why it matters |
| --- | --- | --- | --- |
| Login UX — Auth0 Universal Login vs embedded screen | Click | This review | Most visible user-facing consequence of the auth decision. |
| Inngest scope confirmation (repoint-only vs per-job review) | Click | This review | Materially changes US5 effort. |
| Approval to proceed on the substrate | DEOP | This review | US1/US3 can start immediately against the 001 substrate. |
| ~~Cutover downtime, Auth0 production region, p95 baseline~~ | DEOP | → **004** | Deferred to the Azure-implementation spec with the rest of the real-Azure/cutover scope. |

*Settled (27 Jul 2026): keep Row-Level Security in the database, repointed at the application session.*
