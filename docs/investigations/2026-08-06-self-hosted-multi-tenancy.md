# Self-hosted multi-tenancy — evidence that a third deployment shape exists

**Date:** 2026-08-06
**Scope:** whether a multi-tenant self-hosted deployment is a real shape of the platform, what enables it today, and which assumptions in the carried-over control-plane ADR depend on it not existing.
**Repos examined:** LiarbirdServer (`14d9e9fb`, 2026-08-05 — appliance chart, Helm templates), AgileFramework (`2fae214`, 2026-07-29 — M3/M4 architecture documents, ADRs, known-issues archive), liarbird-docs (`2971843`, 2026-07-28 — internal deployment and how-it-works pages)
**Method:** treat "the shape exists" as a claim needing independent corroboration, and look for it in four places that fail independently — product planning, the commercial layer, the licence mechanism, and deployed chart configuration. A shape is only called *reachable* when configuration in the repo can turn it on without a code change. Then read the control-plane ADR against that finding to locate which of its arguments rest on the tenant count of a self-hosted deployment.

## TL;DR

**A multi-tenant self-hosted deployment is not hypothetical, and it is reachable today in the proof-of-concept.** The appliance chart carries `multiTenant.enabled` and templates `MULTI_TENANT_MODE` into all four service deployments; the appliance licence carries `multi_tenant_enabled` and `max_tenants`, enforced by `liarbirdctl`; AccountManagement models MSSP as a first-class deployment model. Turning it on is a values flip plus a licence, not a code change (§2).

This repo's standing constraints name two deployment shapes — SaaS multi-tenant and **single-tenant** self-hosted. The third is unacknowledged, and the carried-over [control-plane ADR](../adr/control-plane-tenant-plane-separation.md) depends on it not existing in at least five places. The load-bearing one is §2.2, whose storage asymmetry — self-hosted gets one database and two schemas, SaaS gets database-per-tenant — is justified by *"self-hosted has exactly one tenant"* (§4).

The failure mode if that premise is wrong is not a missing feature. It is that a multi-tenant appliance under §2.2 as written has **no isolation mechanism at all**: §2.7's principle is *"the database a row is in is its scope"*, and with N tenants in one `tenant` schema the schema no longer identifies a tenant, while §2.6 puts only four control-plane tables under RLS on the explicit reasoning that tenant-plane tables need no discriminator (§4.2).

The cause is structural rather than an oversight: `DEPLOYMENT_SHAPE` is one variable doing two jobs — who operates the deployment, and how many tenants it holds. Every conditional that a third shape appears to demand comes from that conflation (§5).

Two secondary findings: the appliance backup routine is hardcoded to a single database and would silently omit every tenant under database-per-tenant (§6.1), and the entitlement contract that would gate tenant count is self-contradictory across the two documents that define it (§3).

## 1. What the standing constraints say

This repo's `CLAUDE.md` records two deployment shapes as settled:

> **Two deployment shapes.** SaaS multi-tenant with isolated per-tenant databases, and single-tenant self-hosted. Both are first-class; neither is a degraded mode of the other.

The word doing the work is *single-tenant*. Everything below is evidence that the platform already has a third combination, and that it is the one the constraint excludes by adjective rather than by argument.

## 2. Four independent signals, and one that is decisive

### 2.1 Product planning

`AgileFramework/docs/architecture-m3-multi-tenant.md:838` — `ADR-MT-008`, "License-Based Multi-Tenancy Enforcement" — states the intent directly:

> Multi-tenancy is a licensed feature. VM appliances default to single-tenant mode. Tenant creation is hard-limited by license entitlements.

Its licence-mode table names three modes, of which the middle one is a self-hosted appliance holding up to ten tenants:

| Mode | `multi_tenant_enabled` | `max_tenants` |
|---|---|---|
| Single-Tenant | `false` | `null` |
| Multi-Tenant Limited | `true` | `10` |
| Multi-Tenant Unlimited | `true` | `-1` |

The commercial driver is Epic M4-1, MSSP Multitenant Management Console, listed **Committed** and described as a *"commercial blocker"* (`AgileFramework/docs/m4-future-capabilities.md:13`).

### 2.2 The licence mechanism is an appliance mechanism

`multi_tenant_enabled` and `max_tenants` are fields of the **signed JWT entitlement certificate** defined by `ADR-VM-004` (`AgileFramework/docs/architecture-m3-vm-appliance.md:241`, schema at `:314-315`), validated in Rust by `liarbirdctl` on the VM (`:373-388`). This is the decisive point of provenance: a SaaS control plane has no use for a KMS-signed licence file it would issue to itself. The entitlement exists because the box it runs on is not operated by Liarbird.

### 2.3 The commercial layer models it

`liarbird-docs/internal/deployment/mssp.md` names two MSSP shapes and describes the second as *"MSSP on self-hosted appliance — an MSSP runs their own self-hosted appliance with `MULTI_TENANT_MODE=true` and operates downstream customer tenants on it."* AccountManagement carries `ProductType.LIARBIRD_MSSP` and `DeploymentModel.MSSP`, and its metering collector skips self-hosted MSSP subscriptions specifically because *"self-hosted reports usage through certificate refresh"* — the licence path of §2.2, wired end to end.

### 2.4 It is reachable today

`liarbird-docs/internal/deployment/mssp.md` (`last_verified: 2026-05-03`) records the shape as blocked:

> **Future-proofing only.** Not offered to customers, not comprehensively tested. Cannot deploy today: the application code reads `MULTI_TENANT_MODE` and the licence carries `multi_tenant_enabled` / `max_tenants`, but the appliance Helm chart at `appliance/charts/liarbird/` never sets the env var on any service deployment (ISSUE-268).

**That blocker was resolved the following day.** ISSUE-268 is Resolved 2026-05-04e (`AgileFramework/docs/known-issues-archive.md:1779`), by a five-file chart change. Verified against the chart as it stands:

- `appliance/charts/liarbird/values.yaml:98-99` declares `multiTenant.enabled: false`.
- `MULTI_TENANT_MODE` is templated into all four appliance deployments — `analysis.yaml`, `responder.yaml`, `endpointmgr.yaml`, `dashboard.yaml` (the last also receiving `NEXT_PUBLIC_MULTI_TENANT_MODE`).
- `values.yaml:84-88` documents the three-layer chart / licence / application dependency in place.

So the shape is one values flip and one licence away, with no code change. The internal documentation that says otherwise is stale by a day and has not been re-verified in the three months since.

## 3. The entitlement contract is not actually decided

`max_tenants` has contradictory semantics across the two documents that define it:

| Source | `null` / `None` means | Comparison |
|---|---|---|
| `ADR-MT-008` (Python) | *"use default (1)"*; `-1` is unlimited | `count >= max` |
| `ADR-VM-004` (Rust, `liarbirdctl`) | unlimited — `if let Some(max)` skips the check | `count > max` |

Same field, opposite meaning for the empty value, plus an off-by-one on the limit. Whatever this repo decides about the shape, the entitlement contract is not inheritable as settled.

## 4. What the control-plane ADR assumes

The [control-plane ADR](../adr/control-plane-tenant-plane-separation.md) lists `ADR-MT-008` in its own `Related` header while its decision body assumes that ADR's premise away. Five places depend on a self-hosted deployment having exactly one tenant.

### 4.1 §2.4 contradicts it in a line, with the licence on the wrong side

```
max_tenants: int | None           # 1 self-hosted; from licence in SaaS
```

Self-hosted is hardcoded to 1. The licence is attributed to SaaS, where `ADR-MT-008` sets `max_tenants: -1` and performs no check, while the actual licence enforcement is the appliance's (§2.2). The comment is inverted on both halves.

### 4.2 §2.2's rationale is the load-bearing one

> cross-tenant isolation is the property that matters, and self-hosted has exactly one tenant. Cross-*plane* physical isolation there buys no security while costing real operator complexity.

The entire storage asymmetry rests on that clause. Remove it and the design's own logic turns hostile. §2.7 establishes *"the database a row is in **is** its scope"*; with N tenants sharing one `tenant` schema the schema no longer identifies a tenant. §2.6 places only four **control-plane** tables under `FORCE ROW LEVEL SECURITY`, on the stated reasoning behind CI invariant 5 — *"no `tenant_id`, no RLS requirement"* — because tenant-plane rows take their scope from the connection. A multi-tenant appliance under §2.2 as written therefore has no connection boundary, no schema boundary, no RLS, and no guaranteed `tenant_id` column to fall back on.

This is the shape sold to MSSPs, where the ADR's own §3.2 notes the stakes: *"tenants-within-tenants, compounding blast radius."*

### 4.3 §2.3 is the mechanical blocker

`ALTER ROLE liarbird_tenant SET search_path = tenant, public` binds one role to one schema. Schema-per-tenant in self-hosted needs either per-connection `search_path` — rejected in §2.3 for pooler-trackability, and further constrained by §5's prohibition on PgBouncer `track_extra_parameters` over CVE-2025-12819 — or one role per tenant, which composes with the `(user, database)` pool keying §5 already relies on but turns `StaticDatabaseProvider` from *"~40 lines, no credential indirection"* into per-tenant role and credential provisioning.

### 4.4 §2.5 assumes shape is declared; the licence is refreshable

§2.5 requires deployment shape to be explicitly declared, fail-closed, resolved once at startup and injected, with *"nothing re-derives deployment shape from `os.getenv`"*. `ADR-MT-008` derives tenancy mode from a licence entitlement, and `ADR-VM-004` gives that licence a 24-hour refresh, a 7-day grace period and then an enforcement mode — so `multi_tenant_enabled` can change under a running box. Separately, `ADR-MT-008`'s middleware *assigns* `request.state.tenant_id = DEFAULT_TENANT_ID` when multi-tenancy is off, the soft-fallback class §2.5 exists to convert into loud failures.

### 4.5 §9's reversal condition is self-undermining

> If first revenue turns out to be appliance sales rather than SaaS, the cross-tenant security case largely evaporates — single-tenant deployments have no cross-tenant risk by construction.

Appliance-first revenue via MSSPs and resellers is the most likely route to multi-tenant appliances. The condition would fire on a premise its own trigger falsifies, deferring the isolation work that scenario most needs.

### 4.6 §3.3 loses one leg of three

*"Runtime credential indirection … buys nothing when there is exactly one possible answer"* is void at ten answers. The `pg_dump` atomicity and external-database `CREATEDB` arguments are unaffected, so the conclusion — schemas rather than databases, for a single-tenant deployment — survives on the remaining two.

§5's connection budget is also SaaS-only (`services × replicas × (1 + active tenants) × pool size`) and is not costed for an appliance with a fixed `max_vcpus: 8`.

## 5. The structural cause: one axis doing two jobs

`DEPLOYMENT_SHAPE` conflates **who operates the deployment** with **how many tenants it holds**. Each mechanism in the ADR keys cleanly on one or the other, never on the pair:

| Mechanism | Keys on | Reference |
|---|---|---|
| Storage topology / provider | tenancy | §2.2, §2.4 |
| Sentinel `DEFAULT_TENANT_ID` | tenancy | §2.5 |
| Tenant #1 origin — seed vs provision | tenancy | §2.12 |
| Licence gate on tenant count | hosting | `ADR-MT-008`, `ADR-VM-004` |
| Provisioning credential requirement | storage mode — containerised vs external | §2.4, §3.3 |
| Update, backup, support access | hosting | `ADR-VM-005`, `ADR-VM-006` |

Three combinations are valid: `saas × multi`, `self_hosted × single`, `self_hosted × multi`.

Two observations follow. First, the third combination already runs: §2.12 describes SaaS local development as *"landing with one tenant, one tenant database, one org, and one login"* — containerised Postgres, `ContainerDatabaseProvider.create_database()`, database-per-tenant, one box. That is self-hosted multi-tenant in all but name, and `ADR-MT-001` defines `ContainerDatabaseProvider` as *"CREATE DATABASE via superuser connection to containerized PostgreSQL"*, making the appliance its most native target and SaaS-on-Cloud-SQL the awkward fit.

Second, several ADR sections get *simpler* under two axes: §2.5 becomes "the sentinel exists iff single-tenant" rather than "only in self-hosted", and multi-tenant self-hosted inherits the SaaS rule of zero tenants at install. §2.9's plane assignment — the one the ADR flags as expensive to reverse — is unaffected, and §2.3's role defaults need no edit, since `search_path = tenant, public` resolves to `tenant` under the schema topology and to `public` under database-per-tenant.

## 6. Two costs that are not optional

### 6.1 Appliance backup is hardcoded to one database

`VM-PATTERN-006` (`AgileFramework/docs/architecture-m3-vm-appliance.md:1165`) dumps Postgres as:

```rust
"pg_dump", "-U", "liarbird", "-d", "liarbird", "-Fc"
```

A single logical dump of a single database. Under database-per-tenant this captures the control plane and **silently omits every tenant** — §3.3's `pg_dump` objection, present in code rather than in argument. Cluster-level backup is the answer, and the routine is not single-purpose: the same function dumps Neo4j immediately afterwards via `dump_neo4j` (`:1199-1202`), so the graph's fate and the tenant-coverage defect land in one place.

### 6.2 External-database mode and multi-tenancy are mutually exclusive

Database-per-tenant requires `CREATEDB` on a customer-managed instance, which §3.3 rejected on the grounds that *"many enterprise DBAs will not grant it."* The combination is unreachable rather than merely expensive, so a customer cannot have both bring-your-own-database and multi-tenancy.

## 7. Questions this leaves open

Neither is settled here. Both are open at the time of writing.

1. **Is a multi-tenant self-hosted appliance in scope for the rebuild?** A standing-constraint question before an ADR question, and cheap to answer. A "no" must be stated in the constraint rather than left implicit in an adjective, and it cancels the appliance half of Epic M4-1 — a commercial call, not an architectural one.
2. **If yes, is in-place single→multi upgrade supported?** Multi-tenancy's existence does not touch the single-tenant topology; a licence-driven upgrade path does. §2.9's plane assignment is identical across topologies, so the migration is a schema dump into a new database rather than a data-model change — but only if control-plane and tenant-plane objects keep the same schema names regardless of database packaging. §2.2 currently specifies *"all objects in `public`"* for SaaS tenant databases and leaves the SaaS control database's internal schema unstated.

## 8. Primary artifacts

- `appliance/charts/liarbird/values.yaml:84-88, :98-99`; `appliance/charts/liarbird/templates/deployments/{analysis,responder,endpointmgr,dashboard}.yaml`
- `AgileFramework/docs/architecture-m3-multi-tenant.md:838` (`ADR-MT-008`), `:504` (`ADR-MT-001`)
- `AgileFramework/docs/architecture-m3-vm-appliance.md:241, :314-315, :373-388` (`ADR-VM-004`), `:1165` (`VM-PATTERN-006`)
- `AgileFramework/docs/m4-future-capabilities.md:13`; `AgileFramework/docs/epics.md:87` (`ADR-M3-003`)
- `AgileFramework/docs/known-issues-archive.md:1779` (ISSUE-268, Resolved 2026-05-04e)
- `liarbird-docs/internal/deployment/mssp.md` (`last_verified: 2026-05-03`); `liarbird-docs/internal/how-it-works/multi-tenant-isolation.md` §2
- [`adr/control-plane-tenant-plane-separation.md`](../adr/control-plane-tenant-plane-separation.md) §§2.2–2.6, 2.9, 2.12, 3.2, 3.3, 5, 9
