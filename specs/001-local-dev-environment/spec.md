# Feature Specification: Local Development Environment (Azurite + Docker Postgres)

**Feature Branch**: `001-local-dev-environment`

**Created**: 2026-07-28

**Status**: Draft

**Input**: Re-specification of feature 001. Replaces the previous "Orbit Azure Compute Host" scope
entirely — Azure hosting and Terraform/IaC moved to a separate team and a later Azure-implementation
spec (004). Feature 001 now covers only a **local, cloud-free developer substrate**: a Docker-based
stack (PostgreSQL 17 + Azurite) that any developer can bring up on their own laptop with one command,
mirroring the target Azure-native data/storage engines. It provisions the *engines only* — an empty
PostgreSQL 17 with the required extensions and a working Azurite Blob service. Loading the production
schema (migration replay + Supabase-internal adaptation) and switching the app onto these engines are
the data-access migration's job (spec 002), which is built and validated against this substrate.

## Clarifications

### Session 2026-07-28

- Q: Does 001 replay the production schema, or does it only stand up the empty engines? → A: Engines
  only. 001's Definition of Done is: both containers up healthy, PostgreSQL 17 reachable with pgvector
  / pg_trgm / pgcrypto installed, Azurite Blob create/put/get/delete working, and a one-command
  connectivity smoke that proves all of this offline. The 283-migration replay and its Supabase-internal
  adaptation (auth/storage schema shims, roles, RLS) are **spec 002**, because that work is inseparable
  from the data-access rewrite that will actually target the schema. 001 hands 002 a clean, reproducible
  empty substrate.
- Q: Which PostgreSQL major version should the local engine run? → A: PostgreSQL 17 — matches the
  current source (`supabase/config.toml` `major_version = 17`) so the 002 replay lands faithfully, and
  Azure Database for PostgreSQL Flexible Server supports 17 with pgvector / pg_trgm / pgcrypto; the
  Azure-implementation spec (004) targets the same major version.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Bring the local stack up with one command (Priority: P1)

A developer on a fresh machine runs a single command and gets a running local substrate that mirrors
the target Azure-native stack: a PostgreSQL 17 engine (standing in for Azure Database for PostgreSQL
Flexible Server) with the required extensions enabled, and an Azurite Blob service (standing in for
Azure Blob Storage). No Azure account, no Supabase, no cloud access required (the images are pulled
once, then the stack runs offline).

**Why this priority**: This is the whole point of the feature. Nothing downstream — the schema
migration (002), provider work (003) — can be built until the local engines exist and every developer
can reproduce them. It is the minimum viable slice: the substrate itself.

**Independent Test**: On a machine with only Docker and Node installed and the two container images
already pulled, run the documented bring-up command with no cloud access; confirm PostgreSQL is
reachable with all three extensions present, and that the Azurite Blob endpoint answers.

**Acceptance Scenarios**:

1. **Given** a clean machine with Docker running and no cloud credentials, **When** the developer runs the one-command bring-up, **Then** a local PostgreSQL 17 engine and an Azurite Blob service come up healthy (the command blocks until the healthchecks pass).
2. **Given** the PostgreSQL engine is up, **When** its extensions are inspected, **Then** pgvector, pg_trgm, and pgcrypto are all installed and usable.
3. **Given** the container images are present and no cloud access, **When** the whole bring-up runs, **Then** it completes without reaching Azure or Supabase (the only network step is the one-time image pull).

---

### User Story 2 - Prove the substrate is usable through the target clients (Priority: P2)

With the local engines up, a developer confirms the substrate is genuinely usable through the same
clients the Azure target will use: a direct connection to local PostgreSQL succeeds and the three
extensions are present, and a storage create/put/get/delete cycle succeeds against Azurite through the
Azure Blob client. This proves the wiring and the emulated services are faithful enough for the 002
data-access migration to target.

**Why this priority**: A substrate the application layer cannot actually reach is not useful. Proving
connectivity through the real clients (a live PostgreSQL connection + real Blob operations) is what
makes the environment worth developing 002 and 003 against.

**Independent Test**: With the local stack up and `.env.local` active, run the smoke: connect to local
PostgreSQL and assert the three extensions plus a trivial query succeed with no cloud access; perform a
Blob create/put/get/delete cycle and confirm each succeeds against Azurite.

**Acceptance Scenarios**:

1. **Given** the local stack is up with `.env.local`, **When** the smoke connects to PostgreSQL, **Then** the connection succeeds, the three extensions are present, and a trivial query returns without reaching Supabase or Azure.
2. **Given** the local stack is up, **When** the storage client creates a container and performs a put / get / delete of an object, **Then** each operation succeeds against the Azurite Blob service through the same interface the Azure target will use.

> Note: there is no AI-backed path in 001 (no app runs here), so Anthropic-key behaviour is not
> exercised in this feature — the key is an optional, unused config placeholder locally and is
> exercised by specs 002/003.

---

### User Story 3 - Reset and reproduce a clean environment (Priority: P3)

A developer can tear the environment down and bring it back up to an identical clean state, and any
teammate can go from a fresh clone to a running stack by following the documentation, so the
environment is durable, repeatable, and safe to experiment against on every developer's machine.

**Why this priority**: Reproducibility and easy onboarding are what make a local environment worth
maintaining. Without a clean reset, local state drifts and "works on my machine" problems return; this
story is what lets *every* developer spin the stack up reliably.

**Independent Test**: Bring the stack up, write some data, run the teardown/reset, then bring it up
again and run the smoke; confirm the environment returns to the same known clean state. Separately,
follow the quickstart from a fresh clone and confirm a running stack.

**Acceptance Scenarios**:

1. **Given** a running stack with data written into it, **When** the teardown/reset runs, **Then** all local data is cleared and the next bring-up returns to the same known clean state.
2. **Given** a fresh clone on a new machine, **When** a developer follows the quickstart, **Then** they reach a running, smoke-passing stack without a cloud account or VPN.
3. **Given** repeated bring-up and teardown cycles, **When** each cycle completes, **Then** the outcome is identical (idempotent), verified by re-running the smoke.

---

### Edge Cases

- A required host port (PostgreSQL or the Azurite Blob port) is already in use → bring-up fails with a clear message and the ports are configurable to avoid the collision.
- Docker is not installed or not running → the tooling reports the prerequisite clearly rather than failing obscurely.
- A required PostgreSQL extension is missing from the chosen base image → bring-up fails loudly during init rather than silently producing an incomplete engine.
- The local PostgreSQL major version does not match the intended Azure target version → the mismatch is surfaced so engine parity is not silently broken.
- The Anthropic key is absent → the environment still comes up fully; there is no AI-backed path in 001, so nothing depends on it here (live AI is a 002/003 concern).
- A partial or interrupted bring-up leaves stale volumes → the teardown/reset recovers to a clean state without manual volume surgery.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The environment MUST bring up a local PostgreSQL 17 engine and a local Azurite Blob service with a single documented command, requiring only Docker and Node on the host.
- **FR-002**: The local PostgreSQL engine MUST run PostgreSQL 17 — matching the current source (Supabase `major_version = 17`) and the intended Azure Database for PostgreSQL Flexible Server target (004) — with the pgvector, pg_trgm, and pgcrypto extensions installed and verifiable.
- **FR-003**: The environment MUST expose a local Blob endpoint (Azurite) that supports creating a container and performing put, get, and delete of an object.
- **FR-004**: The environment MUST provide local connection/configuration wiring — a committed `.env.local.example` copied to `.env.local` — that points tooling (and, later, the migrated app (002)) at the local PostgreSQL and Azurite instead of Supabase or Azure.
- **FR-005**: The environment MUST provide a smoke check that verifies, in one run: PostgreSQL reachable with all three extensions and a trivial query (`SELECT 1`); and Azurite Blob create/put/get/delete succeeds — using the same clients the Azure target uses. (Schema replay and app boot-and-serve are spec 002 — see Clarifications.)
- **FR-006**: Teardown and re-bring-up MUST be idempotent — repeated cycles return the environment to the same known clean state with no manual volume surgery.
- **FR-007**: Once the two container images are present, the environment MUST run entirely offline on a developer laptop (macOS/Linux) with no cloud credentials, no VPN, and no runtime reachability to Azure or Supabase. The only network step is the one-time image pull; nothing at runtime reaches a cloud backend.
- **FR-008**: The environment MUST allow the local service ports to be configured so a bring-up can avoid collisions with other local services.
- **FR-009**: The environment MUST be documented with a one-command bring-up, a reset/teardown, and a quickstart that any developer can follow from a fresh clone.
- **FR-010**: Local secrets MUST live only in `.env.local` (dev-only values, never production secrets); a real Anthropic key MUST be optional and needed only to exercise live AI calls.
- **FR-011**: The environment MUST mirror the target Azure-native stack (stock PostgreSQL + Blob storage), not the current Supabase platform, so that code developed against it (002/003) targets the same engines it will run against in Azure.

### Key Entities *(include if feature involves data)*

- **Local PostgreSQL engine**: The Dockerized database standing in for Azure Database for PostgreSQL Flexible Server; empty, with the three extensions installed. (The production schema arrives via the 002 replay.)
- **Local Blob service (Azurite)**: The storage emulator standing in for Azure Blob Storage; holds development objects in local containers.
- **Compose stack definition**: The single declarative definition that brings both services up together and wires their ports/volumes/healthchecks.
- **Local configuration (`.env.local`)**: The set of local, dev-only connection values that point tooling (and later the app) at the local engines.
- **Smoke check**: The single verification run that proves the engines are reachable and healthy through the target clients.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A developer brings the full local stack up from clean with a single command in under 10 minutes on a fresh machine.
- **SC-002**: The smoke passes — PostgreSQL reachable with all three extensions and a trivial query, and Blob create/put/get/delete all succeed against the local storage emulator through the same clients the Azure target uses — with zero cloud dependency (verifiable with all external cloud access blocked; container images already pulled).
- **SC-003**: Teardown followed by re-bring-up returns the environment to an identical clean state, confirmed by the smoke passing both times.
- **SC-004**: A new developer goes from a fresh clone to a running, smoke-passing stack by following the quickstart in under 30 minutes, with no cloud account and no VPN.

## Assumptions

- **Target-stack mirror, not Supabase.** The local substrate emulates the Azure-native target (stock PostgreSQL + Blob), because the application is being moved off Supabase; a Supabase-in-Docker setup would emulate the wrong stack.
- **Engines only; the schema comes with 002.** 001 stands up empty engines with extensions. The 283-migration replay, its Supabase-internal adaptation (auth/storage schema shims, roles, RLS), seed data tied to that schema, and the supabase-js→data-access rewrite are all spec 002 — built and validated against this substrate. Today's supabase-js data access cannot target plain PostgreSQL, so there is no useful app-against-local step to deliver here.
- **PostgreSQL version parity.** The local PostgreSQL major version is 17, matching the current source (`major_version = 17`) and the Azure target (004), so the 002 replay and query behaviour are faithful.
- **Identity is not emulated locally.** Login/identity is a hosted service used via a development tenant; it is not part of this local substrate.
- **The cloud AI provider path is not emulatable locally.** Only the Anthropic path is exercisable locally (using a real key, optionally); the Azure AI Foundry path (003) requires a real cloud endpoint and is validated in the cloud.
- **Developer host baseline.** Developers have Docker and Node installed on macOS or Linux; the environment does not assume any cloud tooling, credentials, or VPN.
- **This environment is shared substrate.** Specs 002 and 003 are developed against it locally, and the future Azure spec (004, owned by a separate team) later mirrors it in the cloud.

## Out of Scope (Deferred)

Explicitly not part of this feature (tracked elsewhere):

1. Any real Azure resources, Terraform/IaC, Azure Container Apps, or ACR — owned by a separate team, delivered as the Azure-implementation spec (004). The prior "Orbit Azure Compute Host" analysis for 001 is archived for that work.
2. **The production schema on the local engine** — the 283-migration replay, Supabase-internal adaptation (auth/storage schema shims, required roles, RLS), and schema-bound seed data. This is inseparable from the data-access rewrite and is delivered as spec 002, built against this substrate.
3. The application code migration itself (data-access rewrite, identity integration, security-policy re-pointing, storage-client swap) and the app booting/serving against the substrate — spec 002.
4. The Azure AI Foundry provider path — spec 003; not locally emulatable.
5. Login/identity flows against a hosted identity tenant — not emulated in the local substrate.
6. Production hardening (TLS, managed secrets, private networking, observability) — handled in the Azure-implementation spec (004).
