# Feature Specification: Azure Landing & Production Cutover (STUB)

**Feature Branch**: `004-azure-landing`

**Created**: 2026-07-29

**Status**: Stub — placeholder holding the deferred Azure/cutover scope. To be fleshed out (via
`/speckit-specify` → `/speckit-plan` → `/speckit-tasks`) by the **separate Azure-implementation team**.

**Input**: Collects the real-Azure and production-cutover scope deferred out of the 001 rescope and the
002 rescope (2026-07-29). Seeded by `_archive/001-azure-host` (the archived "Orbit Azure Compute Host"
analysis). The point of 001 (local substrate) and 002 (app migration on that substrate) is that the
code they deliver runs **unchanged** on Azure — 004 provisions the real Azure and cuts production over.

## Why this spec exists

- **001** = local dev substrate (Docker PostgreSQL 17 + Azurite) — shipped.
- **002** = the application migration off Supabase (Drizzle, Auth0 dev tenant, RLS repoint, Blob
  client, Inngest), built and validated against the 001 substrate — in progress.
- **004** = provision **real Azure** and cut **production** over. This is where every "real Azure
  resource" and "production posture" item lives.

## Scope (deferred here from 001 and 002)

### A. Azure infrastructure as code (Terraform)

- Resource Group; VNet + subnets (delegated PG subnet, private-endpoint subnet).
- `azurerm_postgresql_flexible_server` — private access, Entra auth enabled, extensions allow-listed
  (`pgvector`, `pg_trgm`, `pgcrypto`).
- Storage Account + container + private endpoint (public blob access disabled).
- Key Vault + secrets (Auth0 client secret, service keys, `TOKEN_ENCRYPTION_KEY`).
- User-assigned Managed Identity + role assignments (PG login, Blob Data Contributor, Key Vault
  Secrets User).
- Auth0 **production** tenant as code (`auth0` Terraform provider): region (**US** + EU→US transfer
  safeguard — DPA/SCCs/Data Privacy Framework), application, connections, actions, Mgmt API client.
- `terraform fmt -check` + `validate` CI gate.

### B. Production security posture (Constitution III, in full)

- App→PostgreSQL via **Entra Managed Identity** (no stored password).
- App→Blob and App→Key Vault via Managed Identity.
- Private endpoints for PG + Blob; public exposure limited to the app front door.

### C. Schema + data landing on Azure

- Run 002's `scripts/local/db-replay.sh` (the `$DATABASE_URL`-parameterized shim+replay) against the
  Azure PG Flexible Server — identical mechanism, Azure target.
- Data sync Supabase→Azure with per-table **row-count + checksum** parity (zero rows lost).
- Transfer the real storage bucket objects with pre/post **checksum** verification.

### D. Cutover, rollback, decommission

- **Parallel-run cutover**: keep Supabase live and authoritative until Azure passes the smoke +
  isolation suites; cutover = flip app config to Azure.
- **Rollback** = flip config back to Supabase (cheap — greenfield, no users; nothing lost).
- Retain Supabase read-only N days, then decommission; dependency audit → no Supabase component
  remains in the deployed stack.
- Choosing the Azure **compute host** (App Service vs Container Apps) — the original
  `_archive/001-azure-host` analysis feeds this.

### E. Performance

- Capture p95 query-latency **baseline on Supabase before the freeze**; re-measure on Azure post
  cutover → no regression.

## Success Criteria (to be formalised)

- Production traffic served from Azure (original target: 15 Aug 2026 — revisit given the 001/002
  rescope).
- Zero rows lost (row-count + checksum parity pre/post).
- Isolation suite (from 002) green on Azure pre-cutover.
- No Supabase component remains in the deployed stack.
- 47 Inngest functions healthy for 48h post-cutover.

## Dependencies

- **002 complete** — the app runs green on the 001 substrate (Drizzle, Auth0, RLS, Blob, Inngest, zero
  `supabase-js`). 004 lands that same code on Azure; it does not re-do the app migration.
- Constitution `Orbit on Azure` v1.1.0 binds **in full** here (this is the production end-state) — no
  phased deferrals remain.

## Out of Scope

- The application-layer migration itself (Drizzle rewrite, Auth0 integration, RLS repoint, storage
  client, Inngest repoint) — spec 002.
- The local dev substrate — spec 001.

---

> **This is a stub.** It exists so the deferred Azure/cutover scope is captured and traceable, not
> lost in the 001/002 rescopes. The Azure-implementation team owns fleshing it into a full spec.
