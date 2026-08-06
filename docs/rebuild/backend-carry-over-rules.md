# Backend carry-over rules

What has to change in backend code carried across from the proof-of-concept. Each rule is a
structural property the rebuild requires and the proof-of-concept does not have, so a file carried
across unchanged breaks it — usually silently, because the property is one nothing currently checks.

The dashboard is carried across under no structural conditions at all, deliberately
([`dashboard-carry-over.md`](dashboard-carry-over.md) §1). Nothing here applies to it.

These are not suggestions about style. Most of them fail a build: they restate the fitness functions
in [`../adr/architecture-style.md`](../adr/architecture-style.md) §4.2 pointed at the code someone is
about to paste. Where no check exists, the rule says so rather than borrowing authority it does not
have.

The five modules these rules place code into, and the plane each sits in, are the table in
[`../adr/architecture-style.md`](../adr/architecture-style.md) §4.

Capability-specific guidance — what a particular capability must do, reproduce, or avoid — lives in
[`capabilities/`](capabilities/). This document is what applies across all of them.

## How a rule reads

Three parts, in this order: **the property** the rebuild requires, **what carried code does** that
breaks it, and **the shape it takes instead**. Each closes with what enforces it.

## 1. Connections are acquired in one place

**The property.** Exactly one component acquires a database connection. Feature modules receive a
session; they never fetch one.

**What carried code does.** 42 of the 72 database-touching files in `endpointmgr` reference the
global client or session factory directly. The pattern generalises past that module — a service
holding its own client and lazily initialising it on first use (`analysis/alert_storage.py:47-51`) is
the same defect in a different register.

**The shape instead.** Connection acquisition sits in `platform`, behind the provider seam that
already absorbs both deployment shapes (`shared/tenant/providers/`, `connection_manager.py`,
`dependencies.py`). A carried service takes the session as a parameter. One component already does
this correctly and is the model to copy: the alert processor is stateless with respect to
connections, and every public method takes an `AsyncSession` from its caller
(`endpointmgr/services/alert_processor.py:65-72`).

This is the rule that keeps plane assignment revisable. Whether a table sits in the control plane or
the tenant plane is a change at the chokepoint if there is one, and a change at every call site if
there is not.

**Enforced by** [`../adr/architecture-style.md`](../adr/architecture-style.md) §4.2 assertion 4.

## 2. Queries live in the repository layer

**The property.** ORM and SQL queries are issued by repositories. Routers and services call
repositories.

**What carried code does.** `endpointmgr`'s API modules issue 81 `select()` calls directly, and 6 of
its 36 API modules use the repository layer at all. The layer is not missing — it exists, it is
clean, and it is bypassed. Services do it too: the alert processor holds an `AlertRepository` and
still builds its own filtered query alongside it (`endpointmgr/services/alert_processor.py:396-462`).

**The shape instead.** `shared/repositories/` is the pattern that survives, distributed per module
(§7). Where a repository lacks the method you need, add the method — inlining the query is how the
layer eroded the first time.

**Enforced by** [`../adr/architecture-style.md`](../adr/architecture-style.md) §4.2 assertion 7,
which asserts that `select(`, `update(` and `delete(` appear nowhere outside a module's repository
package. Worth knowing why it exists: every individual violation is a two-line convenience, which is
how the layer eroded the first time.

## 3. A module reaches another module only through its public surface

**The property.** A feature module imports only `<module>/api.py` of another. `platform` imports no
feature module at all.

**What carried code does.** The clearest violation is a dependency inversion: the licence gate lives
in the shared layer and imports a service out of `endpointmgr` at four sites
(`shared/middleware/license_gate.py:118,191,257,365`), so every application mounting the gate
acquires the whole module. The cross-service import edges mostly leave with the descope; this one
does not, because both ends are staying.

**The shape instead.** Note that moving the gate's import target to the `entitlements` module does
not fix it — `platform` may not import a feature module either. So either the check is injected at
application wiring time, or the gate belongs to `entitlements` rather than to `platform`. Settle it
when that module lands. Do not settle it with an import.

**Enforced by** §4.2 assertions 1 and 2.

## 4. Every table has exactly one owning module

**The property.** Each ORM table is declared by one module, and table names stay globally unique.

**What carried code does.** All 48 tables are declared in the shared layer — 39 in
`shared/db_models.py`, 5 in `shared/tenant/db_models.py`, 4 in `shared/forwarding_models.py`. No
module owns a table, which is the property that made independent deployment impossible irrespective
of process count.

**The shape instead.** The declaration moves next to the module that owns the data;
[`../adr/architecture-style.md`](../adr/architecture-style.md) §4 gives the destinations. Tables
orphaned by the descope do not come across at all — the `forwarding_*` set, and the rareness and
statistics tables. Global uniqueness is preserved so that schema-per-module stays available later as
organisation, without schema-qualifying application queries.

**Enforced by** §4.2 assertion 3.

## 5. Background work is not started by the request-serving role

**The property.** Schedulers, pollers and long-lived tasks are registered by the `scheduler` role.
The `web` role starts none.

**What carried code does.** Seven schedulers and workers start in the request-serving lifespan —
cleanup, uninstall expiry, agent reaper, pending-response expiry, token expiry, session history,
deprovisioning (`endpointmgr/main.py:363-473`) — alongside an alert-forwarding task that leaves with
the descope (`:339`). The production overlay runs that same process at `minReplicas: 2` with
autoscaling enabled, so the reaper and the expiry sweeps are configured to run in duplicate. No
tenants in production is the only reason this is armed rather than firing.

**The shape instead.** Registration moves behind a module-level runner that the `scheduler`
entrypoint calls, so the same code is reachable from a role that is not replicated. Any job that must
survive replication anyway claims its rows: `FOR UPDATE SKIP LOCKED` is the one existing precedent
(`shared/services/response_queue.py:1082-1114`), and `tasks/scheduled_cleanup.py:111-112` — which
selects pending rows with no claim at all — is the shape not to copy, including as a template.

**Enforced by** §4.2 assertion 5.

## 6. Code lands inside a module, not in a flat tree

**The property.** Carried code arrives inside one of the five modules, grouped by domain.

**What carried code does.** `endpointmgr` spans roughly fifteen domains as 36 API modules and 36
service modules with no grouping between them, one of which is 2,656 lines
(`endpointmgr/api/agent_registration.py`). It is what a monolith with conventional structure becomes,
and it is the least navigable component in the estate.

**The shape instead.** Most of it lands in `endpoints`, which is deliberately unsplit — but landing
*in* `endpoints` is not landing *as* a flat directory. Subdividing it internally needs no ADR
([`../adr/architecture-style.md`](../adr/architecture-style.md) §4.3), so group on the way in, where
it is free, rather than after 40 files have arrived.

**Enforced by** [`../adr/architecture-style.md`](../adr/architecture-style.md) §4.2 assertion 8,
which fails on a source file outside a declared module root. That catches the flat tree and nothing
more — whether a module's internal grouping is sensible is not expressible as a check, so the second
half of this rule is a review obligation. A file-length ceiling in the linter is the cheap partial
proxy.

## 7. The disposition of `shared`

`shared` is 90 files and 35,047 lines — the second-largest component in the estate and not a service.
It does not carry across as a unit. It splits three ways:

| Destination | What goes there |
|---|---|
| `platform`, as a layer | Middleware (JWT, RBAC, rate limiting), audit, telemetry, connection acquisition, and the tenant provider seam that already absorbs both deployment shapes |
| The owning feature module | ORM declarations (rule 4), and domain services |
| Nowhere | `graph_notifier.py`, `neo4j_ingestion.py`, `forwarding_*`, and the descoped services' models |

**Repositories are per-module, not a `platform` layer.** This follows from the assertions rather than
being a preference: a repository declares queries against a model, so a `platform`-level repository
would import feature-module models and trip assertion 1. The layer's shape survives; its location
changes.

The licence gate (rule 3) is the one file in `shared` whose destination is genuinely unresolved, and
it is unresolved because `platform` and `entitlements` are both barred from importing each other's
side of it.

## 8. Status

This document is written against
[`../adr/architecture-style.md`](../adr/architecture-style.md), which is **Proposed**. It promotes
when a Phase 1 vertical slice runs with its §4.2 assertions green, and if an assertion changes on the
way through, the rule resting on it changes with it.

Every rule here has an assertion behind it except the second half of rule 6 — whether a module's
internal grouping is sensible — which stays a review obligation because it is not expressible as a
check.

## 9. Related

- [`../adr/architecture-style.md`](../adr/architecture-style.md) — the decision these rules
  implement, including the module table and the fitness functions
- [`../adr/control-plane-tenant-plane-separation.md`](../adr/control-plane-tenant-plane-separation.md)
  — plane assignment, and why grants cannot enforce module boundaries
- [`../investigations/2026-08-05-architectural-style-assessment.md`](../investigations/2026-08-05-architectural-style-assessment.md)
  — the independent cost accounting the counts above corroborate
- [`capabilities/`](capabilities/) — per-capability briefs
- [`dashboard-carry-over.md`](dashboard-carry-over.md) — the dashboard's carry, which these rules do
  not govern
