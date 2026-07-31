# Feature Specification: Real-Time Salesforce Inbound (CDC)

**Feature Branch**: `005-salesforce-cdc-inbound`

**Created**: 2026-07-30

**Status**: Draft

**Input**: User description: "Add real-time Salesforce inbound (CDC) to Orbit: one always-on stream-shipper container + Inngest replay-cursor, gap-recovery backfill, per-record ordering, External-Id provisioning. .NET reference is SALESFORCE-INTEGRATION-ANALYSIS.md §8."

## Overview

Orbit already syncs contacts with Salesforce in both directions, but **inbound** (Salesforce → Orbit) is driven by scheduled polling on an hourly/daily cadence — flagged in code as an interim stand-in for real-time ingestion. This feature replaces that polling with **near-real-time change ingestion** so that a change made in a customer's Salesforce org is reflected in Orbit within seconds, survives outages without losing changes, applies changes in the correct order without duplicates, and sets up each connected org automatically so records match cleanly.

The proven end-to-end design for this exists in the .NET reference implementation and is summarized in `SALESFORCE-INTEGRATION-ANALYSIS.md` (§8 = target architecture, §9 = edge-case catalog). This spec adapts that design to Orbit's stack.

## Clarifications

### Session 2026-07-30

- Q: Should real-time CDC replace the hourly inbound polling, or keep a reconciliation sweep as a safety net? → A: Match the .NET reference — CDC push is the sole steady-state inbound path; there is **no scheduled poll**. Reconciliation is provided only by gap-recovery and manual/on-connect backfill.
- Q: What availability posture for the always-on ingestion process? → A: Single always-on instance with fast restart; the durable cursor makes a restart lossless. No warm standby / leader election in v1.
- Q: How many connected Salesforce orgs must ingestion support at launch? → A: Under 50 orgs — a single instance multiplexing all CDC streams over one shared connection.
- Q: On rollout, retro-stamp already-synced contacts with the new External-Id, or forward-only? → A: Retro-provision — a one-time backfill stamps the External-Id on all currently-linked contacts.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Near-real-time inbound reflection (Priority: P1)

A marketer using Orbit relies on contact data that matches Salesforce. When a colleague updates a Lead or Contact in Salesforce (name, email, phone, mapped custom field, or deletes the record), that change appears in Orbit within seconds, not on the next hourly poll — so segments, journeys, and outreach act on current data.

**Why this priority**: This is the headline value and the reason the feature exists. Without it, inbound data can be up to an hour (or a day) stale. It is the minimum viable slice: real-time reflection of Salesforce edits is demonstrable on its own.

**Independent Test**: Connect a Salesforce developer org, edit a Contact's phone number in Salesforce, and confirm the change is visible on the matching Orbit contact within the freshness target — with no manual "Sync now" click.

**Acceptance Scenarios**:

1. **Given** an org with real-time inbound active, **When** a user edits a mapped field on a Salesforce Contact, **Then** the corresponding Orbit contact reflects the new value within the freshness target.
2. **Given** an org with real-time inbound active, **When** a Salesforce Lead is created that maps to a new Orbit contact, **Then** a matching Orbit contact is created (or an existing one is linked) within the freshness target.
3. **Given** an org with real-time inbound active, **When** a Salesforce record is deleted, **Then** the matching Orbit contact link is severed per the configured deletion behavior, and the contact itself is retained.
4. **Given** a mapped field is direction-restricted to outbound-only or is a protected consent field, **When** that field changes in Salesforce, **Then** Orbit does **not** overwrite the local value.

---

### User Story 2 - No silent data loss across outages (Priority: P2)

An operator needs confidence that inbound sync never drops changes, even when the ingestion process restarts, deploys, or is offline for an extended period. On short interruptions, ingestion resumes exactly where it left off. On interruptions longer than Salesforce's change-retention window, the system detects the gap and runs a full reconciliation backfill instead of silently skipping the missed changes.

**Why this priority**: A CRM sync that silently loses records is worse than no sync — it erodes trust in the data. This is the reliability backbone that makes real-time inbound safe to depend on. It builds directly on US1 but is independently testable.

**Independent Test**: With ingestion active, stop the ingestion process, make several Salesforce edits, restart the process, and confirm all edits land (short-outage resume). Separately, simulate an outage longer than the retention window and confirm a gap-recovery backfill runs and reconciles the affected records rather than skipping them.

**Acceptance Scenarios**:

1. **Given** ingestion is briefly interrupted, **When** it restarts, **Then** it resumes from the last acknowledged position and applies every change made during the interruption exactly once.
2. **Given** ingestion is offline longer than Salesforce's change-retention window, **When** it restarts, **Then** the system detects the expired position, triggers a full reconciliation backfill, and does not silently skip the gap.
3. **Given** the ingestion process crashes mid-batch, **When** it recovers, **Then** no change is lost and no change is applied twice (re-delivery is safe).
4. **Given** a backfill is already running for an org, **When** another trigger fires (manual, gap recovery, or reconnect), **Then** the system coalesces to a single backfill rather than running overlapping ones.

---

### User Story 3 - Automatic org setup on connect (Priority: P2)

An admin connecting Salesforce should not have to hand-configure Salesforce metadata for sync to work. On connect (and on demand for repair), Orbit provisions the prerequisites in the customer's org: a stable external-identifier field on Lead and Contact, the change-event channel Orbit listens to, and the field-level permissions the integration user needs. The admin sees whether setup succeeded, and if it failed (e.g. insufficient permissions), a clear, actionable status with a retry.

**Why this priority**: Real-time ingestion depends on the change-event channel existing, and clean, duplicate-free matching depends on the external-identifier field. Automating this removes a fragile manual step and is the difference between "works in a demo org" and "works for every customer." It can be tested independently of live sync by verifying the org metadata after connect.

**Independent Test**: Connect a fresh Salesforce developer org and confirm that the external-identifier field, the change-event channel with Lead/Contact members, and the required field permissions are created and verified — and that a status surface reports readiness. Re-running provisioning on an already-provisioned org is a safe no-op.

**Acceptance Scenarios**:

1. **Given** a newly connected org, **When** provisioning runs, **Then** the external-identifier field, change-event channel (with Lead and Contact members), and field permissions are created and verified, and readiness is reported.
2. **Given** provisioning was already completed, **When** it runs again, **Then** it recognizes existing objects and completes without creating duplicates or erroring.
3. **Given** the connecting user lacks permission to modify org metadata, **When** provisioning runs, **Then** the failure is captured as an actionable "setup incomplete" status with a retry, not a silent or misleading success.
4. **Given** provisioning is not yet ready, **When** a sync would run, **Then** it is held with a clear reason rather than failing obscurely.

---

### User Story 4 - Correct ordering and no duplicates per record (Priority: P3)

When the same Salesforce record changes several times in quick succession — or is updated and then deleted — Orbit must end in the state that matches Salesforce's final state, never applying an older change after a newer one and never creating duplicate contacts from concurrent processing.

**Why this priority**: Ordering and idempotency are correctness guarantees that matter under load and at the edges (rapid edits, update-then-delete, re-delivery). They harden US1/US2 rather than adding new user-visible capability, so they are lower priority but still required for a trustworthy sync.

**Independent Test**: Rapidly update a Salesforce record several times and then delete it; confirm Orbit ends in the deleted-link state (not stuck on an intermediate value). Concurrently deliver the same change twice; confirm exactly one effect and no duplicate contact.

**Acceptance Scenarios**:

1. **Given** multiple changes to one record arrive close together, **When** they are processed, **Then** the final Orbit state matches Salesforce's final state regardless of processing concurrency.
2. **Given** an "update then delete" sequence for one record, **When** processed, **Then** the delete is not overtaken by the earlier update.
3. **Given** the same change is delivered more than once, **When** processed, **Then** it produces exactly one effect (no duplicate contact, no double-applied edit).
4. **Given** two changes match the same contact by email while identifiers are stale, **When** processed, **Then** the system holds the ambiguous case for review rather than merging or overwriting the wrong contact.

---

### Edge Cases

- **Delete events carry no field data** → matching a delete relies on the stored Salesforce identifier only; a delete for a record Orbit never synced is recorded as an informational no-op, not an error.
- **No email / duplicate email inbound** → a Salesforce record with no email is still ingested (matched by identifier); a change whose email matches more than one Orbit contact, or matches one already bound to a different Salesforce record, is held for review, never auto-merged.
- **Echo loop** → a change Orbit itself wrote to Salesforce must not bounce back inbound and re-trigger an outbound push; sync-originated writes are suppressed from re-triggering the opposite direction.
- **Consent / opt-out protection** → subscription and opt-out fields are never overwritten by inbound changes (compliance-critical), even under a misconfigured mapping.
- **Ingestion process unavailable** → changes accumulate at Salesforce for the retention window; short outages resume losslessly, long outages trigger gap recovery (US2).
- **Salesforce API limits** → ingestion and any backfill respect the org's shared API budget and back off rather than exhausting it.
- **Token expiry / reconnect required** → an expired or revoked authorization moves the org to a "reconnect required" state and pauses ingestion for that org without affecting others; dormant orgs are kept alive so their authorization does not idle-expire.
- **Malformed / undecodable change event** → a single bad event is retried a bounded number of times, then skipped and recorded, so one poison event cannot stall an org.
- **First connect** → connecting does not trigger an automatic bulk import of the entire org; existing records are linked (reconcile-only) and a full import is an explicit, on-demand action.

## Requirements *(mandatory)*

### Functional Requirements

**Ingestion & freshness**
- **FR-001**: The system MUST ingest Salesforce Lead and Contact change events (create, update, delete) for every connected org in near-real-time, without waiting for a scheduled poll.
- **FR-002**: The system MUST apply an ingested change to the matching Orbit contact within the freshness target under normal conditions (see SC-001).
- **FR-003**: The system MUST discover which orgs have Salesforce connected and maintain live ingestion for exactly those orgs, starting ingestion when an org connects and stopping it when an org disconnects.
- **FR-004**: The system MUST honor the org's configured sync direction and per-field direction, applying inbound changes only for fields that permit inbound writes.
- **FR-023**: Once real-time ingestion is active, the system MUST NOT run a scheduled inbound poll; steady-state inbound is CDC push only, and the sole reconciliation mechanisms are gap recovery (FR-007) and manual/on-connect backfill (FR-008), matching the .NET reference. The existing hourly/daily poll MUST be retired as part of this feature.

**Durability & recovery**
- **FR-005**: The system MUST persist a per-org ingestion position (cursor) and resume from it after any restart, deploy, or crash.
- **FR-006**: The system MUST advance the stored cursor only after the corresponding change(s) have been durably accepted for processing, so that a crash results in safe re-delivery rather than a lost change.
- **FR-007**: The system MUST detect an expired or invalid ingestion position (outage longer than Salesforce's change-retention window) and trigger a full reconciliation backfill instead of silently resuming past the gap.
- **FR-008**: The system MUST provide a resumable backfill that reconciles an org's records, invokable manually ("sync now"), on gap recovery, and on reconnect; and MUST coalesce concurrent triggers into a single running backfill per org.
- **FR-009**: Connecting an org MUST NOT trigger an automatic full import; first-connect reconciliation MUST link existing records without importing the whole org, and a full import MUST be an explicit action.

**Ordering & idempotency**
- **FR-010**: The system MUST process changes for the same Salesforce record in order, so a newer change is never overwritten by an older one.
- **FR-011**: The system MUST make change application idempotent, so re-delivery of the same change (from recovery or overlap) produces exactly one effect.
- **FR-012**: The system MUST match an inbound change to an Orbit contact by Salesforce identifier first and email second; MUST self-heal a stale identifier; and MUST hold (not auto-merge or overwrite) ambiguous email matches.
- **FR-013**: The system MUST treat a delete as a link-sever matched on the stored Salesforce identifier only, retaining the contact record; a delete for a never-synced record MUST be an informational no-op.

**Provisioning**
- **FR-014**: On connect (and on demand), the system MUST provision in the customer's org: a stable external-identifier field on Lead and Contact, the change-event channel Orbit listens to (with Lead and Contact members), and the field-level permissions the integration user requires.
- **FR-015**: Provisioning MUST be idempotent (safe to re-run) and MUST classify failures as transient (retryable) versus permission-denied (actionable "setup incomplete"), surfacing status the admin can see and retry.
- **FR-016**: The system MUST hold sync with a clear reason when provisioning is not verified ready, and MUST re-verify readiness when a subsequent operation indicates the org configuration has drifted.
- **FR-024**: On rollout for an already-connected org, the system MUST run a one-time backfill that stamps the external identifier on all currently-linked contacts, so external-id matching is clean for the existing base (not only for records that change after go-live). This backfill MUST be idempotent and resumable like any other backfill (FR-008).

**Protection & safety**
- **FR-017**: The system MUST never overwrite subscription/opt-out (consent) fields via inbound changes, regardless of field-mapping configuration.
- **FR-018**: The system MUST prevent an echo loop: a change Orbit wrote to Salesforce MUST NOT re-trigger an outbound push when it returns inbound.
- **FR-019**: The system MUST respect the org's Salesforce API budget during ingestion and backfill, backing off rather than exhausting it, and MUST keep a connected-but-idle org's authorization from idle-expiring.
- **FR-020**: An expired or revoked authorization for one org MUST move that org to a "reconnect required" state and pause only that org's ingestion, without affecting other orgs.

**Observability**
- **FR-021**: The system MUST record inbound sync outcomes — runs, per-record errors/holds, and backfill results — so an operator can answer "what synced, what failed, and why" for a given org.
- **FR-022**: A single undecodable or repeatedly failing change MUST be retried a bounded number of times, then skipped and recorded, without stalling the org's ingestion.

### Key Entities *(include if feature involves data)*

- **Ingestion cursor**: Per-org marker of the last acknowledged change position; the basis for lossless resume and gap detection.
- **Inbound change event**: A normalized record of one Salesforce change (entity, change type, record identifier, changed fields, commit time, replay position), decoupled from the transport format.
- **Contact↔Salesforce link**: Per-contact sync state — Salesforce identifier, object type (Lead/Contact), last-synced time, and a sever marker for deletions.
- **Sync run / backfill run**: A record of an ingestion or reconciliation run with status and counts.
- **Sync error/hold**: A per-record failure or held-for-review case with reason and enough snapshot detail to diagnose after the fact.
- **Provisioning status**: Per-org readiness of the external-identifier field, change-event channel, and permissions, with a verified-at timestamp.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: For an active org under normal conditions, a Salesforce change is reflected in Orbit within **60 seconds** (target p95), versus the current up-to-1-hour polling latency.
- **SC-002**: Across an ingestion restart or deploy, **100% of changes** made during the interruption are applied, with **zero** duplicates and **zero** lost changes (verified by reconciliation count).
- **SC-003**: After an outage longer than the change-retention window, gap recovery reconciles the affected records so the post-recovery Orbit state matches Salesforce with **no unexplained missing records**.
- **SC-004**: For a burst of rapid changes to a single record (including update-then-delete), the final Orbit state matches Salesforce's final state in **100%** of test cases.
- **SC-005**: Connecting a fresh Salesforce org results in verified provisioning (identifier field, change-event channel, permissions) in a **single connect flow**, with a clear pass/fail status and a working retry on failure.
- **SC-006**: **Zero** consent/opt-out fields are ever changed by an inbound sync in the compliance test suite.
- **SC-007**: A single malformed change event never blocks an org's ingestion — the org continues processing subsequent events, and the bad event is recorded.
- **SC-008**: A single ingestion instance sustains real-time inbound for the full launch scale (up to ~50 connected orgs) while holding the SC-001 freshness target, with no scheduled inbound poll running.

## Assumptions

- **Reuses existing Orbit Salesforce foundation**: OAuth connect/token refresh/disconnect, the provider-generic CRM mapping/routing/inclusion layer, the outbound write path, per-field system-of-record conflict resolution, and contact storage already exist and are reused rather than rebuilt. This feature adds the inbound-real-time path and its supporting cursor, gap recovery, ordering, and provisioning.
- **A persistent ingestion process is required**: Salesforce change ingestion uses a long-lived streaming connection that cannot run inside a purely serverless/request-scoped function. The design assumes **one small always-on process** dedicated to holding the stream, decoding events, checkpointing the cursor, and emitting each change to Orbit's existing durable workflow engine (Inngest); all downstream apply/backfill/ordering logic stays in that engine. Per the project constitution (Azure-native, managed-first), this process targets **Azure Container Apps** (a single always-on replica), not a VM.
- **Single-instance ingestion (confirmed)**: One ingestion instance per environment, with fast restart; the durable cursor makes a brief ingestion pause lossless, so warm standby / leader election is out of scope for v1.
- **Launch scale (confirmed)**: Under 50 connected orgs at launch — a single ingestion instance multiplexing all CDC streams over one shared connection is sufficient; sharding across instances is out of scope for v1.
- **Freshness target**: "Near-real-time" is defined as p95 < 60s end-to-end; sub-second is not required.
- **Scope is inbound-only**: Outbound (Orbit → Salesforce) already works and is out of scope except where echo-loop suppression and shared provisioning touch it.
- **Object scope**: Lead and Contact only, matching current mapping support; other Salesforce objects are out of scope.
- **UI ownership**: Any admin-facing surfacing of provisioning/sync status is built to the backend contract and handed to the UI team (Click) per existing ownership conventions; this spec covers the behavior and the data, not the visual design.
- **Reference design**: `SALESFORCE-INTEGRATION-ANALYSIS.md` §8 (target architecture) and §9 (edge-case catalog) are the authoritative blueprint; the proven .NET implementation is the reference for every behavior above.

## Dependencies

- A connected Salesforce org with the OAuth scopes needed to read records and provision metadata (external-identifier field, change-event channel, permissions).
- Orbit's existing durable workflow engine (Inngest) and its Supabase data store for cursor, links, runs/errors, and provisioning status.
- A hosting target capable of running one always-on process (Azure Container Apps per the constitution).
- Salesforce's change-data-capture / streaming capability enabled for the org's edition.
