# Control plane / tenant plane separation with deployment-shaped storage topology

**Status:** Proposed — moves to Accepted once the Phase 1 vertical slice confirms the two gates in §8.1
**Date:** 2026-08-03
**Deciders:** Platform / Engineering
**Scope:** LiarbirdServer (`database/`, EagerBeaver `shared/tenant/`), `liarbird-helm`, `appliance/` (charts + `liarbirdctl`), `liarbird-infrastructure` (`gke-saas/`)
**Related:** ADR-M3-003 (database-per-tenant), ADR-MT-008 (licence-based multi-tenancy), ADR-004 (SaaS tenant provisioning and tenant auth), ADR-VM-008 (database HA with K8s operators)
**Supersedes in part:** the implicit placement decisions in Story M3-1.4 and migration `069`; the role
matrix (`:186-196`) and permission matrix (`:199-212`) in
`AgileFramework/docs/architecture-m3-multi-tenant.md`, **on audit scope only** — those tables grant
cross-tenant audit visibility to four of five audit-viewing roles and are no longer authoritative on
that point (§2.7). They are not being edited; this ADR is the reference.

---

## 1. Context

### 1.1 What exists today

One PostgreSQL schema serves two unrelated jobs. It is both the **tenant registry** — `tenants`,
`tenant_databases`, `dashboard_users`, `usage_metrics` — and the **tenant data schema** —
`agents`, `alerts`, `commands`, `detection_profiles`. Multi-tenancy was retrofitted at migration
`032` of `092`; the product was single-tenant for the first third of its schema life, and
`tenants` arrived as one more table in the existing lineage rather than as a separate plane.

The consequences are visible in three places:

**Every tenant database carries a dead copy of the registry.** `bootstrap_tenant_db` applies
`01-schema.sql` plus all 92 migrations unfiltered, so each tenant DB contains a permanently empty
`tenants`, `tenant_databases`, `tenant_groups`, `tenant_group_members`, `agent_tenant_mapping`,
`dashboard_users`, `user_sessions`, `user_permission_overrides`, `usage_metrics`,
`registration_tokens`, and `audit_logs`. Nothing writes to them. The reverse also holds — the
platform DB carries full `agents`/`alerts`/`commands` tables that should be empty in SaaS.

**The SQL is physically duplicated.** `database/init/` + `database/migrations/` is vendored
byte-identically to `EagerBeaver/src/shared/tenant/bootstrap/sql/`, because the endpointmgr Docker
build context is `EagerBeaver/`, which cannot see the canonical tree. Parity is enforced by
`EagerBeaver/tests/unit/tenant/test_bootstrap_sql_parity.py`; the promised
`scripts/sync_bootstrap_sql.py` helper never landed, so every new migration is a manual two-place
copy caught after the fact by red CI.

**Isolation is enforced two different ways with no principle distinguishing them.** Tables reached
via `get_tenant_db()` are protected by the connection. Tables reached via `get_platform_db()` are
protected by a `WHERE` clause — as that dependency's own docstring concedes: *"Be careful not to
leak data between tenants. Always filter by tenant_id when returning results to non-admin users."*

### 1.2 Why this matters now

**The filter-protection class has already produced a cross-tenant defect.** ISSUE-219, filed as a
SaaS-launch production blocker:

> LangGraph internal `agent_id` resolution lookups query the platform DB without a tenant filter,
> so an alert from tenant A can resolve to tenant B's agent record on a hostname or UUID match.

It was found and fixed. The class remains open, defended only by review vigilance.

**Degrading to the default tenant fails in two opposite directions, and only one of them is safe.**
`DEFAULT_TENANT_ID` (`00000000-0000-0000-0000-000000000001`) is injected by
`TenantContextMiddleware` in single-tenant mode, and used as a *soft fallback* in at least three
places that do not check the mode — `analysis/enrichment.py:71`,
`shared/middleware/rate_limiter.py:77`, `analysis/rareness_service.py:68`. There are two distinct
paths by which a SaaS request can end up carrying the sentinel, and they behave differently:

| Path | Behaviour today | Why |
|---|---|---|
| **Routing** — `get_tenant_db()` selects a connection | **Fails closed.** `_load_credentials` returns `None`, `get_connection_string` raises `tenant_database_not_found` | `037_seed_default_tenant.sql` inserts into `tenants` but *not* `tenant_databases`, so there is no database to route to |
| **Filtering** — `WHERE tenant_id = …` on a control-plane table | **Fails open.** Reads return the sentinel's rows (empty in a clean SaaS deploy); **writes succeed** | `037` *does* seed the `tenants` row, so the sentinel satisfies the FK on `registration_tokens`, `dashboard_users`, `usage_metrics`, and `audit_logs` |

The fail-closed half is frequently mistaken for the whole picture. It is not, and it is accidental:
no test asserts the absence of that `tenant_databases` row, nothing documents it as
security-relevant, and the natural "improvement" of giving the default tenant a real database
converts it silently to fail-open.

The fail-open half is live today. Its blast radius: a write that degrades lands real rows under a
tenant nobody owns, with no error, no constraint violation, and nothing anomalous in the audit trail.
Two different tenants degrading in the same deployment commingle their data under one identity. Reads
degrade to the silent-empty-result hazard — a customer sees an empty list rather than a 500, which
reads as "no data" rather than "broken."

`endpointmgr/api/installer.py:143` hard-guards exactly one call site against the sentinel. That is
evidence the hazard was recognised, not that it was closed.

**The mode itself defaults to the unsafe value.**
`MULTI_TENANT_MODE = os.getenv("MULTI_TENANT_MODE", "false")` (`middleware.py:46`), read across
**40 references in 13 files** — `enrichment.py`, `jwt_auth.py`, `installer.py`, `agents.py`,
`license_service.py`, `langgraph_agent.py`, `pending_response_expiry.py`, `responder/main.py`.

So the failure is not per-request. A missing or misspelled env var in one Helm values file puts an
entire service into single-tenant mode, where it stamps the sentinel on **every** request it handles,
in a deployment where other services are correctly multi-tenant. No startup check rejects this. The
one adjacent guard makes it worse rather than better: `responder/main.py:54-68` reads the same
variable to decide how strictly to enforce `GCP_LOCATION`, so the same dropped variable also
downgrades a data-residency check from fatal to a warning.

**The window is open and will close.** Per `docs/investigations/2026-07-31-deployment-shapes-audit.md`,
GKE SaaS prod has been externally live since 2026-06-28 **with zero tenants**. Staging holds demo
tenants only and is burnable. Neo4j and Redis are not used by active features and can be excluded
from a rebuild. Doing this work after the first real tenant means a cross-database data migration
on partitioned tables under a change freeze.

---

## 2. Decision

Adopt a **control plane / tenant plane** split, with the storage topology of each plane determined
by deployment shape, and defence-in-depth isolation appropriate to each plane.

### 2.1 Two planes, two migration lineages — plus a seed category

| Plane | Holds |
|---|---|
| **Control** | tenant registry, tenant DB credentials, platform users and sessions, licensing, usage metering, registration tokens, platform-scope audit |
| **Tenant** | agents, alerts, commands, detection profiles, enumerations, deployment manifests, forwarding, responses, tenant-scope audit |

This is the control-plane / application-plane separation that AWS's SaaS architecture fundamentals
and Azure's multitenant guidance both treat as foundational — the tenant plane here is what those
documents call the application plane.

Two independent lineages with independent `schema_migrations` tables. No scope tags, no conditional
application within a lineage.

**A third category exists and is not a lineage: shape-specific seeds.**

```
database/
  control/migrations/      applied in both shapes
  tenant/migrations/       applied in both shapes
  seeds/self-hosted/       applied only when DEPLOYMENT_SHAPE=self_hosted
```

The default-tenant seed lives here rather than in either lineage, because SaaS must not have a
sentinel (§2.5). **It does not go in `database/init/`**, and the reason is worth recording because it
is the obvious first suggestion:

- `init/` is mounted at `/docker-entrypoint-initdb.d` (`docker-compose.yml:91`), so Postgres runs it on
  a virgin data directory — *before* any migration. `037_seed_default_tenant.sql` could never have gone
  there in the current schema: `init/01-schema.sql` contains no `tenants` table (it arrives at
  migration `032`), so the insert would fail at container init.
- More decisively, **`init/` does not execute in real deployments at all.**
  `database/Dockerfile.migrations` copies only `migrations/*.sql`, and the Helm hook runs that image.
  `init/` has exactly two consumers: the compose Postgres entrypoint, and `bootstrap_tenant_db` reading
  `_INIT_DIR / "01-schema.sql"` directly. So a seed placed there would silently never run in SaaS or on
  the appliance.

`migrations/` was therefore the only directory that ran everywhere, which is why `037` is a migration.
That was a correct call given the constraints, not a filing error. The new `seeds/` directory is
applied by an explicit applier in every deployment shape, which is the property `init/` cannot provide.

**Bycatch worth fixing while collapsing the history (§2.11):** `init/01-schema.sql` (33,441 bytes) and
`migrations/000_base_schema.sql` (28,861 bytes) are two independently-maintained base schemas ~4.5 KB
apart, reaching different deployments by different paths. That is a third duplication alongside the
vendored copy, and a plausible source of "works locally, fails in staging." After the rebuild each
lineage has one base schema and `init/` stops being a schema source.

### 2.2 Storage topology varies by deployment; application code does not

| | Control plane | Tenant plane | Operator sees |
|---|---|---|---|
| **Self-hosted** | database `liarbird`, schema `control` | database `liarbird`, schema `tenant` | **one** database, one connection string, one `pg_dump` |
| **SaaS** | database `control` | database `tenant_<slug>` × N, all objects in `public` | control plane + N tenant DBs |

In both cases the application holds two engines. Queries stay unqualified; resolution happens via
`search_path`. This is viable without touching any SQL: **zero schema-qualified references exist**
across `01-schema.sql` and all 92 migrations (verified 2026-08-03).

Rationale for the asymmetry: cross-tenant isolation is the property that matters, and self-hosted
has exactly one tenant. Cross-*plane* physical isolation there buys no security while costing real
operator complexity. SaaS separates planes *and* tenants at the database level, so the guarantee
lands where it is needed.

### 2.3 `search_path` is set at the role level, not per connection

```sql
ALTER ROLE liarbird_control SET search_path = control, public;
ALTER ROLE liarbird_tenant  SET search_path = tenant,  public;
```

Role defaults are applied by Postgres at session establishment, so they are part of the connection's
initial state rather than something a pooler must track. Extensions (`uuid-ossp`, `pg_trgm`,
`btree_gin`) are per-database and live in `public`, hence `public` last on both paths.

These two roles need **different default `search_path`, not different object ownership** — both can
be members of the same owning role, so nothing about the CNPG or Cloud SQL topology changes.

### 2.4 Two providers behind the existing seam

`TenantDatabaseProvider` (3 abstract methods) already exists, and `TenantConnectionManager` already
exposes a settable `provider` property.

- `ContainerDatabaseProvider` (existing) — SaaS. `CREATE DATABASE`, credentials generated and stored
  encrypted in `tenant_databases`.
- `StaticDatabaseProvider` (new, ~40 lines) — self-hosted. Connection string from config,
  `CREATE SCHEMA IF NOT EXISTS`, `DROP SCHEMA`. No runtime `CREATE DATABASE`, no credential
  indirection, no `CREATEDB` grant required in external-database mode.

The seam also bounds future divergence: a tier requiring stronger placement — instance- or
project-per-tenant, as the §3.2 compliance pathway may eventually demand — is a third provider
implementation, not a change to this decision. The trigger is the first tenant contractually
requiring data residency outside `australia-southeast1`. At that point control-plane placement and
the Zitadel instance need deciding as well, since §2.9 holds `dashboard_users`, `user_sessions`,
`user_permission_overrides` and `user_terms_acceptance` in the single control plane.

Policy divergence that cannot sit behind the storage seam gets one explicit home:

```python
@dataclass(frozen=True)
class DeploymentProfile:
    provider: TenantDatabaseProvider
    default_tenant_id: UUID | None    # None in SaaS — no sentinel exists
    max_tenants: int | None           # 1 self-hosted; from licence in SaaS
    allow_tenant_drop: bool
```

Resolved once at startup and injected. Nothing re-derives deployment shape from `os.getenv`.

### 2.5 The sentinel exists only in self-hosted

`DEFAULT_TENANT_ID` is coherent only where "exactly one tenant" is an invariant. In SaaS it must not
exist: no seed row, no configured value, and the service refuses to start if one is present
alongside multi-tenancy.

SaaS deploys with **zero** tenants. Tenant #1 arrives via the normal signup/provisioning path, the
same as tenant #47. The first platform admin comes from Zitadel (`zitadel-init`, `04-zitadel.sh`),
not the database.

Consequence: the soft fallbacks in `enrichment.py`, `rate_limiter.py`, and `rareness_service.py`
become loud failures in SaaS instead of writes to a real tenant.

**Deployment shape has no default value.** This is the other half of the fix, and it does not follow
from removing the sentinel. Today `MULTI_TENANT_MODE` defaults to `"false"` (§1.2), so an absent
variable is indistinguishable from a deliberate single-tenant deployment — and a guard of the form
"refuse to start if a sentinel exists alongside multi-tenancy" never fires, because the service does
not know it was supposed to be multi-tenant.

`DeploymentProfile` (§2.4) is therefore resolved from **explicitly declared** configuration with no
fallback: if deployment shape is absent or unparseable, the service refuses to start rather than
assuming either shape. Fail-closed on configuration, not fail-quiet into the more permissive mode.

### 2.6 Row-Level Security on the control plane

Tenant-scoped tables that must remain in the control plane are protected by RLS with
`FORCE ROW LEVEL SECURITY`, not by a lint. After §2.7 moved tenant audit to the tenant plane, that
set is: `registration_tokens`, `usage_metrics`, `agent_tenant_mapping`, and `dashboard_users`.

These are small and mostly admin-facing, so RLS here is less load-bearing than when audit was in
scope — but it still covers the one defect class that has actually occurred. ISSUE-219 was an
agent-resolution leak through the platform DB, i.e. `agent_tenant_mapping`, which remains on this
list.

This is the bridge model: silo for the tenant plane, pool + RLS for the control plane. It gives the
control plane the same *class* of protection — database-enforced, not application-enforced — that
the tenant plane gets from the connection.

`FORCE` is required because the application role owns the tables it queries. This is the documented
solve for exactly that case, on both our platforms:

- Postgres: *"Table owners normally bypass row security as well, though a table owner can choose to
  be subject to row security with `ALTER TABLE … FORCE ROW LEVEL SECURITY`."*
- Google Cloud SQL: *"If you want to subject a table owner to row-level security, then use the
  `ALTER TABLE … FORCE ROW LEVEL SECURITY` command."*
- CNPG sets the application user as a **low-privilege, non-superuser owner** of the application
  database, so `FORCE` binds it.

Note this does **not** contradict migration `069`'s finding that GRANT/REVOKE is ineffective for
owners. GRANT/REVOKE has no override; RLS has `FORCE` precisely for this.

**Tenant context is bound per transaction, never per session:**

```sql
SELECT set_config('app.tenant_id', :tenant_id, true);   -- third arg = is_local
```

`set_config(..., true)` rather than literal `SET LOCAL`, because `SET` is DDL-like and cannot take
bind parameters — this also removes an injection path. Transaction-scoped settings are reset by
Postgres at commit/rollback, so no cleanup step exists to forget. SQLAlchemy's
`pool_reset_on_return='rollback'` default provides a second layer. This is the same mechanism
PostgREST and Supabase use (`SET LOCAL request.jwt.claims`), chosen specifically for
pooling-compatibility.

The binding is issued from the session factory / FastAPI dependency, so no handler can obtain a
control-plane session without tenant context already bound. This puts isolation enforcement in one
shared mechanism rather than in per-handler discipline — the posture the AWS SaaS Lens prescribes
(isolation should not be left to service developers, but applied by a shared mechanism outside
their view), and the inversion of the "always filter by tenant_id" docstring model that produced
ISSUE-219.

**Platform-admin access is encoded in policy, not in `BYPASSRLS`:**

```sql
CREATE POLICY registration_tokens_tenant_scope ON registration_tokens FOR ALL USING (
     current_setting('app.scope', true) = 'platform_admin'
  OR tenant_id = current_setting('app.tenant_id', true)::uuid
);
```

This avoids depending on a role attribute that may not be grantable on Cloud SQL, and keeps both
access modes inside the policy. The trust boundary becomes "who may set `app.scope`" — the same
application-controlled boundary already required for `app.tenant_id`.

### 2.7 Audit splits by plane — one table per lineage, no scope discriminator

**Tenant audit lives in the tenant plane. Platform audit lives in the control plane.** Each lineage
gets its own `audit_logs` table holding only its own scope. The database a row is in *is* its scope,
so no `scope` column, no `tenant_id IS NULL` convention, and no CHECK constraint are needed.

Consequences:

- Tenant audit becomes **connection-protected** rather than filter- or policy-protected — the
  strongest isolation available, and stronger than the RLS this ADR applies elsewhere.
- GDPR/SOC 2 per-tenant export becomes a single-database operation rather than a filtered extract.
- The `audit_logs_block_mutation()` append-only trigger belongs to **both** lineages. Its superuser
  exemption (for `liarbirdctl/src/retention/postgres.rs`) carries over unchanged.
- `tenant_id` is retained on tenant-plane audit rows for export and merge convenience, but it no
  longer carries scope meaning and is not load-bearing for isolation.
- Archive and export are produced **per lineage, in two shapes**: tenant-plane rows carry `tenant_id`,
  control-plane rows do not (§2.9). One combined archive format would have to synthesise the scope
  column this section exists to avoid.

**Negative decision recorded: a unified cross-tenant audit view is not a requirement — at any tier.**
Confirmed with Product (2026-08-03) as an oversight rather than a design intent. This covers both:

- **Tier 0** (Instance Admin, Instance Support) — no all-tenant audit feed. Support navigates to one
  tenant at a time.
- **Tier 1** (MT Admin, MT Support) — **no unified audit view across an MSSP's assigned tenant group.**
  Asked explicitly and answered no, so the MSSP epic does not inherit a fan-out requirement.

Consistent with the code as it stands: `include_all_tenants=True` exists in
`audit_log_repository.py` but no caller passes it, `endpointmgr/api/audit_logs.py:36` states there is
*"currently no cross-tenant override path,"* and `PLATFORM_ADMIN` appears only in docstrings. The
capability was reserved, never built.

If it is ever wanted, the route is fan-out across the relevant tenant databases — tractable for a
bounded MSSP group, awkward to paginate and sort at platform scale. Dual-writing to a platform table
is *not* an available route (§3.6).

**The boundary of this negative decision is audit, not operations.** The AWS SaaS Lens expects
operations teams to keep tenant-aware views of fleet health, activity, and consumption; nothing here
removes that. Fleet-level operational visibility is served by telemetry that never lived in tenant
databases — `usage_metrics` in the control plane, service metrics and logs in Cloud Logging — so it
is untouched by the audit split and needs no fan-out. What this section rejects is a unified *audit*
view; it is not citable against an ops dashboard or per-tenant health monitoring.

**Supersedes the role and permission matrices in `AgileFramework/docs/architecture-m3-multi-tenant.md`.**
Those tables assert the opposite: the role matrix (`:186-196`) grants all-tenant audit visibility to
Instance Admin and Instance Support, and the permission matrix (`:199-212`) marks *View audit logs* ✅
for MT Admin and MT Support — four of five audit-viewing roles cross-tenant capable. Per this ADR they
are superseded and not authoritative on audit scope. They will not be edited; this document is the
reference. A reader who finds them in conflict should treat this ADR as current.

**Cross-plane events — every event is written to exactly one plane.** An action by a platform-tier
actor on a tenant's data has one audit destination, never two. Dual-writing the same event to both
planes is rejected (§3.6).

The destination rule is keyed on *touched*, not *changed*:

| What the action touched | Audit destination |
|---|---|
| Tenant data — **read or write** | that tenant's `audit_logs` |
| Control-plane data only | control-plane `audit_logs` |

Keyed on **touched** rather than **changed** deliberately: a mutation-shaped rule records nothing
when Instance Support *reads* a tenant's alerts for diagnostics, and vendor read access is exactly
the event customers expect to see audited. This rule also resolves the awkward category correctly —
"Instance Admin flags tenant X for abuse investigation" touches no tenant data, so it lands in the
control plane where the tenant cannot see it.

The rule is total and deterministic: every event touches tenant data or it does not, so there is
always exactly one destination and no event can be dropped or duplicated by ambiguity.

**This holds for platform-actor events too — there is no exception.** An Instance Admin's action on a
tenant's data is recorded in that tenant's audit log and nowhere else.

The obvious objection is insider threat: the actor being recorded holds credentials reaching every
tenant database, so the record sits within their reach. That is true, and it is **not fixed by copying
the event to the control plane** — the control plane is inside the same credential domain, so the same
person has access to both. Two Postgres databases under one operator's control are one trust boundary
with two tables in it, not two boundaries. The existing append-only trigger already concedes the point
by exempting superusers, which is how `retention/postgres.rs` purges `audit_logs` at 365 days as
`postgres`.

The real mitigation is an append-only sink outside the database and outside the operator's delete
authority — GCS with retention lock, Cloud Logging with a retention policy, or forwarding to a
customer-held SIEM. That is a different concern from plane assignment and is **out of scope for this
ADR**; see the follow-up in §9. Note the shell exists but is unwired: `analysis/siem/` has a
forwarder, connectors, and formatters, none of which reference audit.

### 2.8 `registration_tokens` stays in the control plane

Structurally forced: a registration token is what *establishes* tenant context. The agent presents
it before any tenant is known, and `validate_registration_token()` runs first. It cannot be looked
up in the tenant database because the tenant database is not yet known. This is the same reasoning
Story M3-1.4 documents for its sibling `agent_tenant_mapping` — *"avoids querying all tenant DBs to
find which tenant owns an agent; enables O(1) security validation"* — though it is not written down
for the token itself.

Protected by control-plane RLS, as above.

### 2.9 Plane assignment

| Control plane | Tenant plane |
|---|---|
| `tenants`, `tenant_databases`, `tenant_groups`, `tenant_group_members` | `agents`, `agent_network_interfaces`, `agent_groups`, `agent_group_memberships` |
| `agent_tenant_mapping` | `alerts` (+ partitions), `commands`, `pending_responses`, `response_actions` |
| `dashboard_users`, `user_sessions`, `user_permission_overrides`, `user_terms_acceptance` | `detection_profiles`, `custom_tag_patterns`, `policies`, `parsers` |
| `registration_tokens` | `device_enumerations`, `user_enumerations` |
| `usage_metrics` | `deployment_manifests`, `update_packages`, `update_policy`, `update_rollouts`, `agent_version_history` |
| `audit_logs` — **platform-scope events only** (no tenant filtering; see below) | `audit_logs` — **tenant-scope events** (connection-protected, append-only trigger) |
| `tls_certificate_config`, `certificate_rotation` | `forwarding_*`, `integrations`, `system_settings`, `statistics_hourly`, `system_events` |
| | `uninstall_*`, `scheduled_cleanups`, `security_software`, `ai_payload_templates`, `*_profiles` |

`audit_logs` exists in **both** lineages as separate tables with the same shape, each holding only
its own plane's events (§2.7).

**The control-plane `audit_logs` carries no `tenant_id` and no RLS policy.** It is read only by
platform-tier roles, so there is nothing to filter by tenant. Platform events that concern a tenant
without touching its data — "Instance Admin flagged tenant X for abuse investigation" — reference it
via the existing `resource_type='tenant'` / `resource_id` columns added in migration `088`, not via
a `tenant_id` scope column. This keeps invariant 5 (§7) unambiguous: no `tenant_id`, no RLS
requirement.

**Remaining contested tables stay in the control plane** where a filter bug is caught by RLS, rather
than being guessed into the tenant plane. Plane assignment is the only decision here that is
expensive to reverse: moving a table between planes post-launch is a cross-database migration on
live tenants.

### 2.10 Foreign keys crossing the plane boundary become bare UUIDs

Four columns, all of shape "which human did this":

| Table | Column |
|---|---|
| `uninstall_codes` | `created_by → dashboard_users(id)` |
| `uninstall_audit` | `authorized_by → dashboard_users(id)` |
| `certificate_rotation` | `created_by`, `cancelled_by → dashboard_users(id)` |

Precedent exists in the codebase: `agent_tenant_mapping.agent_id` is already a bare
`UUID PRIMARY KEY` with no FK, for exactly this reason. Every FK into `tenants` already originates
from a control-plane table, so no change is needed there.

### 2.11 Collapse the migration history

Replace 92 migrations with two fresh `000_base_schema.sql` files, one per lineage. History remains in
git. This sheds the renames (`039`, `057`), the ISSUE-348 renumbering fallout, the nine collided
version prefixes, and makes tenant provisioning materially faster. Viable only because there are no
production tenants; this is the last moment it is free.

### 2.12 Local development declares a shape and bootstraps tenant #1 automatically

`docker compose up -d` must still yield a working stack with a usable login, in **either** shape. Today
that works because `037_seed_default_tenant.sql` is in the lineage; once it isn't, the seeding has to
move somewhere shape-aware without costing a dev any extra steps.

**The shape is declared, never defaulted — but the declaration ships in the repo.** §2.5 forbids a code
default; it does not forbid a template. `.env.example` carries `DEPLOYMENT_SHAPE=self_hosted`, and
`scripts/generate-secrets.sh` already copies that template on first run. A fresh clone is therefore
zero-config while a misconfigured *deployment* still fails loudly. Compose asserts it too, using the
`:?` form already used for `POSTGRES_PASSWORD`:

```yaml
x-shape-env: &shape-env
  DEPLOYMENT_SHAPE: ${DEPLOYMENT_SHAPE:?DEPLOYMENT_SHAPE must be set in .env}
```

A YAML anchor rather than a repeated literal, because the current file gets this wrong:
`MULTI_TENANT_MODE: ${MULTI_TENANT_MODE:-false}` appears **once** (`docker-compose.yml:284`, on the
analysis service) while twelve other modules read the variable. That is the §1.2 hazard — one service
in a different mode from its neighbours — already live in the dev compose file.

**Bootstrap chain.** One new link on a pattern the file already uses (`service_completed_successfully`
at `docker-compose.yml:231`, `:302`, `:409`):

```
postgres (healthy)
  └─ db-migrate          one-shot, shape-aware: creates schemas, applies lineage(s)
      └─ tenant-bootstrap one-shot, shape-aware: seeds or provisions tenant #1
          └─ backends
```

Two containers rather than one, for the same reason §9 splits the Helm hooks: a failure should say
whether migrations or provisioning broke.

| Shape | `db-migrate` | `tenant-bootstrap` |
|---|---|---|
| **self-hosted** | creates `control` + `tenant` schemas, applies both lineages | applies `seeds/self-hosted/` — the sentinel `tenants` row, `ON CONFLICT DO NOTHING` |
| **SaaS** | applies the control lineage to the control DB | provisions one dev tenant **through `ContainerDatabaseProvider.create_database()`** — the production code path, not SQL |

**Provisioning the dev tenant through the real path is deliberate, and it retires a stated weakness.**
§4 lists as a negative that tenant provisioning is exercised only by SaaS and CI, never the dev loop.
If a developer working in SaaS shape provisions a tenant on every `up`, that path becomes among the
most-travelled code in the repo instead of the least.

**A tenant needs an identity, not just a database.** ADR-001 makes tenant = Zitadel organisation, so
without an org and a member there is no JWT carrying a tenant claim and nothing to exercise. This is
the part that actually determines whether multi-tenant local dev is usable. `zitadel-init/main.py` is
the right home: it is already an idempotent one-shot with `search_project()` → `create_project()`,
gated behind `--profile internal-auth` (`docker-compose.yml:576`, `:666`). Extend it with the same
search-then-create shape for a dev org and dev user, and have it publish the org ID for
`tenant-bootstrap` to adopt as the tenant's identity.

So multi-tenant local dev is `docker compose --profile internal-auth up -d`, landing with one tenant,
one tenant database, one org, and one login.

**Idempotency is required, not optional** — `up` is run repeatedly against an existing volume.
Migrations already track state; the self-hosted seed uses `ON CONFLICT DO NOTHING`; the SaaS path
checks for the dev tenant's slug before provisioning.

**Reset.** `docker compose down -v` remains the clean slate. Worth adding one narrower affordance: a
script that drops and reprovisions a single tenant via the provider (`drop_database()` +
`create_database()`) without discarding Postgres, Neo4j, and Zitadel — the fastest loop for anyone
working on provisioning itself.

Command-level instructions belong in `docs/local-setup.md`, updated when this lands. What this ADR
fixes is that local dev declares a shape and that tenant #1 exists without manual SQL.

### 2.13 A tenant-lineage migration is a fleet operation

In SaaS, applying a tenant-lineage migration means applying it to N databases, and the runner
Phase 2 builds must treat partial failure as the normal case rather than the exception. The
semantics it must have:

- **Per-database progress, convergence by re-run.** Each tenant database's own `schema_migrations`
  is the unit of progress. The runner applies per tenant, records per tenant, and skips what is
  already applied — so the recovery procedure for any mid-fleet failure is "fix, re-run," never a
  manual reconciliation of which tenant got what.
- **Deterministic order, canaries first.** Tenants migrate in a stable order with one or more
  designated canary tenants (internal or demo) at the front, so a systemically bad migration is
  caught at blast radius one. A failure stops the fleet rollout; already-migrated tenants stay
  migrated.
- **A mixed-version fleet is a normal operating state.** Between the first tenant and the last —
  or indefinitely, after a partial failure — some tenant databases run V+1 while others run V.
  Post-launch, tenant-lineage migrations must therefore be expand/contract: additive first,
  destructive only once every tenant database and the code depending on the old shape have moved.
  This is a review-gate rule on migration content, not a CI-checkable invariant.
- **Per-tenant schema version is observable from the control plane.** The runner records each
  tenant database's current version alongside `tenant_databases`, so "which tenants are still on V"
  is a query, not an investigation. A tenant whose database failed mid-migration is marked
  unhealthy and its requests fail closed — the same posture §1.2 establishes for routing failures.

Self-hosted is the N=1 degenerate case of the same runner; nothing shape-specific is needed.

---

## 3. Considered and rejected

### 3.1 Status quo plus a filter lint

A lint asserting that every query against a control-plane table from a non-admin route carries a
tenant predicate would address the ISSUE-219 class for roughly 5% of the effort.

**Rejected as insufficient, not as wrong.** A lint reduces the recurrence rate of a bug class; it
does not make it unreachable. It also cannot see a runtime connection's state. It remains the fallback
if §8.1's `FORCE RLS` gate fails, since control-plane protection then has nothing else to rest on.

### 3.2 Pool model with RLS only — no plane split

**This is the 2026 industry default** and the option most teams would pick: shared schema, `tenant_id`
on every table, RLS as the enforcement layer. It is cheaper, needs no per-tenant bootstrap, no
credential indirection, and gives a trivially simple self-hosted story. Current guidance places it as
correct for *"early-stage products with 100–10,000 tenants and no compliance requirements beyond
SOC 2."*

**Rejected for this product**, on four grounds:

1. We sell endpoint deception **to security teams**; isolation claims are scrutinised in every deal,
   and "enforced by the connection" outranks "enforced by row policy" in a security review.
2. `docs/` in AgileFramework includes a **US government compliance pathway**, where data residency and
   physical isolation are effectively table stakes. Silo keeps that reachable — a tenant's data already
   sits in its own database, so regional placement is a provider change (§2.4) rather than a schema
   migration. This covers the tenant plane; control-plane and identity residency is separate (§2.9).
3. The **MSSP tenant-group** model means tenants-within-tenants, compounding blast radius.
4. ADR-M3-003 already decided silo and `ContainerDatabaseProvider` is built; the marginal cost of
   completing it is lower than the cost of unwinding it.

AWS is explicit that this is a context judgement, not a correctness one: *"these models are all
equally valid… the regulatory, business, and legacy dimensions of a given environment often play a
big role."* We adopt RLS **in addition to** silo (§2.6), not instead of it.

### 3.3 Two physical databases in self-hosted

The obvious reading of "two planes." **Rejected** in favour of two schemas:

- `pg_dump` of one database is an atomically consistent snapshot of both planes. Two databases means
  two dumps in two transactions, and a restore that lands the tenant DB at a different point than the
  control plane's `tenant_databases` credentials row leaves a broken appliance.
- External-database mode would require `CREATEDB` on a customer's Cloud SQL/RDS instance. Many
  enterprise DBAs will not grant it. `CREATE SCHEMA` is a far easier ask.
- Runtime credential indirection — connect to control plane, read a row, connect to the tenant DB —
  buys nothing when there is exactly one possible answer, and adds an opaque failure mode.

Citus 12's schema-based sharding provides precedent for schemas as a legitimate tenancy boundary,
including the observation that it requires no data-modelling changes.

### 3.4 One migration lineage with `-- scope:` tags

Considered while assuming self-hosted needed a single physical plane. **Rejected** as unnecessary once
self-hosted gains a real tenant schema — the tags only existed to work around that constraint.

### 3.5 A reserved "platform" row in `tenants` (platform-as-a-tenant)

Would make `tenant_id` `NOT NULL` everywhere and yield a single uniform RLS predicate.

**Rejected**: it recreates the `DEFAULT_TENANT_ID` hazard in new clothing — a magic UUID that code can
degrade into, where a lost tenant context lands in a real, queryable row rather than erroring. The
split-by-plane audit of §2.7 removes the need for any scope marker at all, magic or otherwise.

### 3.6 Dual-writing audit events to both planes

The alternative to §2.7's single-destination rule: write every cross-plane event to both the tenant's
audit log and the control plane's, so each side has a complete picture without fan-out.

**Rejected**, primarily on atomicity:

- **Two databases means two transactions with no shared commit.** A failure between them leaves the
  audit trail inconsistent, and for an append-only table there is no safe repair — retry risks
  duplicates, and the append-only trigger blocks the `UPDATE`/`DELETE` that would fix them. An audit
  log that can silently disagree with itself is worse than one that is merely incomplete.
- It **couples the two lineages**: the tables must keep compatible shapes forever, reintroducing by
  the back door the cross-plane coupling this ADR removes.
- It **double-counts in exports and reporting**. GDPR/SOC 2 tenant export would need to know which
  copy is authoritative — a distinction with no natural answer.
- The problem it solves — a consolidated platform-side view — is the same fan-out problem the
  negative decision in §2.7 already accepted. Paying a permanent correctness cost to avoid a query
  nobody has asked for is the wrong trade.

**Rejected for platform-actor events too, not merely as a general policy.** The atomicity objection is
a property of writing across two databases, so narrowing the event class does not weaken it. And the
insider-threat motivation for such an exception does not survive scrutiny: both planes sit inside the
same credential domain, so a control-plane copy adds no separation (§2.7). Audit durability against a
privileged insider needs an off-box append-only sink, not a second table — see the follow-up in §9.

### 3.7 Role-based `REVOKE` to enforce the plane boundary

**Rejected on the codebase's own evidence.** Migration `069`: *"GRANT/REVOKE is ineffective when the
application's DB role is also the table owner (verified 2026-05 in the docker-compose dev
environment, and is the typical config in CNPG-managed and Cloud-SQL-managed deployments)."* We also
create zero roles today. Replaced by an assertion (§7).

---

## 4. Consequences

### Positive

- Filter-protection becomes connection-protection for the tenant plane, and database-enforced row
  policy for the control plane. The ISSUE-219 class becomes structurally unreachable rather than
  merely less likely.
- `MULTI_TENANT_MODE` stops being a data-access concern. `get_tenant_db()` never falls back to the
  platform DB, removing the silent-empty-result hazard.
- The sentinel is confined to the deployment where "one tenant" is invariant; the latent fail-open
  path is removed.
- No dead registry tables in tenant DBs; no tenant data tables in the control plane.
- Tenant audit is connection-protected — stronger than the RLS applied elsewhere — and GDPR/SOC 2
  per-tenant export becomes a single-database operation rather than a filtered extract.
- Splitting audit by plane removes the scope-marker problem entirely rather than fixing it: no
  `scope` column, no `tenant_id IS NULL` convention, no three-valued-logic hazard in policies.
- Self-hosted operators manage one database, one connection string, one atomically consistent backup.
- External-database mode needs no `CREATEDB` grant and performs no runtime DDL on customer instances.
- The vendoring problem shrinks — only the tenant lineage needs to reach endpointmgr.
- Self-hosted → SaaS becomes "provision tenant 2."
- The `tenant_databases` credential indirection means SaaS tenant databases need not share one
  Cloud SQL instance: when an instance's connection or capacity budget fills, tenant N+1 provisions
  onto a new instance with no application change. Connections are not the only per-instance budget:
  Cloud SQL collects per-database metrics for at most 500 databases, so observability coverage caps
  tenant count per instance too. Single-instance limits are a provisioning event, not an
  architectural ceiling.
- Collapsing 92 migrations sheds accumulated rename/renumber debt and speeds provisioning.

### Negative

- **Install-flow work is the largest cost.** `037_seed_default_tenant.sql` becomes a provisioning step;
  the `liarbirdctl` wizard's `DatabaseStepData` gains `CREATE SCHEMA` permission validation and
  pre-existing-schema detection; `reset/factory.rs` must drop two schemas (and in external mode, only
  those two). Slow to iterate because the feedback loop is Packer image → VM boot → wizard →
  Helm install.
- **Two provider implementations** — the classic "supports both modes" complexity risk. Mitigated by a
  narrow 3-method interface, a trivially small second implementation, and the enforced invariant in
  §7. Honest framing: this cost is *purchased* by the self-hosted operator simplification in §2.3.
- **The self-hosted plane boundary is convention plus assertion, not grant-enforced**, because `public`
  is on both search paths and grants are ineffective under owner topology. Harmless with one tenant.
- Four FK columns drop to bare UUIDs; cross-plane joins move to the application layer.
- Two connection pools per service instead of one — in self-hosted. In SaaS it is one control-plane
  pool plus one per *active tenant* database, which makes per-tenant pool sizing a real constraint
  (§5), not a footnote.
- Migration runner and Helm hooks must become lineage-aware (two jobs, hook weights `-6`/`-5`, new
  `tenant.bootstrapDefault` value: true for appliance, false for GKE) — and the runner fleet-aware
  (§2.13): canary-first ordering, per-tenant resumability, and expand/contract discipline on
  tenant-lineage migrations post-launch.
- App pods may briefly race ahead of tenant #1 registration on first install (migrations run
  `post-install`). Fail-closed, but readiness must report not-ready rather than treating it as fatal.
- Per-tenant restore in SaaS is a procedure, not a button. Cloud SQL point-in-time recovery restores
  a whole instance, so restoring one tenant means cloning the instance to the recovery point,
  extracting that tenant's database with `pg_dump`, and restoring it into prod. Needs a runbook
  (liarbird-docs) before the first real tenant.
- Tenant provisioning is not exercised by the *self-hosted* dev loop, which uses
  `StaticDatabaseProvider` — weaker coverage than a single-provider design would give. Partly offset by
  §2.12: a developer working in SaaS shape locally provisions through the real
  `ContainerDatabaseProvider` path on every `up`. Residual gap is narrower than first assessed, but real
  for anyone who only ever runs self-hosted shape.
- **Any future cross-tenant audit view becomes a fan-out** across tenant databases rather than one
  query. Accepted deliberately (§2.7); the capability does not exist today and is not required. The
  cost lands on the MSSP epic if a unified group-level view is later wanted.
- **A platform admin's actions on tenant data are audited only in tenant databases**, so the platform
  has no consolidated record of its own staff's activity. Accepted (§2.7): a control-plane copy would
  not improve it, since both planes share one credential domain. Durability against a privileged
  insider is addressed by the off-box export follow-up in §9, not by this ADR.

### Neutral / already true

- Appliance backup needs no change: `backup/manager.rs:814` runs `pg_dump -Fc -U postgres eagerbeaver`,
  a whole-database dump with no `--schema` flag, as `postgres` (a real superuser under CNPG, therefore
  RLS-bypassing). This is currently accidental and becomes an invariant (§7).
- Neither environment deploys a connection pooler today (zero PgBouncer/`Pooler` references). SaaS uses
  Cloud SQL via `cloud-sql-proxy` native sidecar, which is a per-connection IAM/TLS tunnel with no
  multiplexing.

---

## 5. Constraints this decision imposes

**Per-tenant engines must not inherit the global pool defaults.** In SaaS, connection demand is
`services × replicas × (1 + active tenants) × pool size`, against a `max_connections` that is a
memory-tied default when not set explicitly (500 today on prod's `db-custom-4-15360`; the flag can
be raised as the instance scales, though defaults plateau around 800–1,000). At the current
application defaults — `POSTGRES_POOL_SIZE=10`, `POSTGRES_MAX_OVERFLOW=20`, set in no Helm values
file, so every environment runs the hardcoded numbers — four backend services fully subscribe those
500 connections at roughly nine concurrently active tenants. The constraints: tenant-plane engines
get small pools (`pool_size` 2–3, low overflow); idle tenant engines are evicted rather than held
forever; and `POSTGRES_POOL_*` is set explicitly per deployment. Past what pool discipline and
vertical scaling buy, the escape hatch is horizontal: new tenants on a new instance (§4).

**Connection pooling.** No pooler is deployed today and none is proposed, so this is a constraint on a
future change rather than present work. If a PgBouncer `Pooler` is ever added to the appliance:

- Use `poolMode: session`, **or** rely on the two distinct roles from §2.3 — PgBouncer keys pools on
  `(user, database)`, so the planes' pools separate naturally even under transaction pooling.
  **Verify this empirically at that point** (stand up a `Pooler` in transaction mode with both roles
  and confirm the pools stay separate); if it does not hold, `poolMode: session` is mandatory.
- **Never** enable `track_extra_parameters` for `search_path`. `search_path` is not marked `GUC_REPORT`
  in stock Postgres, so PgBouncer cannot track it reliably, and **CVE-2025-12819** (fixed in PgBouncer
  1.25.1) allowed unauthenticated arbitrary SQL execution during authentication via a malicious
  `search_path` in the StartupMessage when that setting was combined with `auth_user` and a
  non-fully-qualified `auth_query`.

The SaaS-side equivalents are Cloud SQL Managed Connection Pooling or an in-cluster PgBouncer, and
the trigger for either is sustained connection count above ~70% of `max_connections` *after* the
pool-sizing constraints above are in place. Known before reaching for them:

- Managed Connection Pooling requires the Enterprise Plus edition and an instance restart to enable.
  Prod runs standard Enterprise, and **nothing else in this ADR needs the upgrade** — the mention here
  is not a pending procurement. Its pools are keyed per (user, database), so the two roles of §2.3
  separate the planes' pools naturally — the same property relied on for PgBouncer above.
- Transaction mode — the default, and the case `set_config(..., true)` and role-level `search_path`
  were chosen for (§2.3, §2.6) — prohibits session state (`SET`/`RESET`, `LISTEN`, session advisory
  locks) and requires `max_prepared_statements` tuning for asyncpg, our driver. Self-managed
  PgBouncer has the same shape: prepared statements need PgBouncer ≥ 1.21 or
  `statement_cache_size=0` in the driver.

**Backup, retention and archive roles must bypass RLS.** `pg_dump` defaults to `row_security = off`,
which errors rather than silently producing a partial dump — but `--enable-row-security` would
silently truncate. The backup role must be superuser or hold `BYPASSRLS`. This aligns with the
existing `audit_logs_block_mutation()` superuser exemption that lets
`liarbirdctl/src/retention/postgres.rs` purge as `postgres`.

Archive is the third reader of these tables and the one where a missing bypass is hardest to notice:
a truncated archive reports success, and the gap surfaces only when the archived rows are needed.
`AgileFramework/docs/compliance/data-retention-and-disposal-policy.md` places an archive tier after
the active window on alert and response data, on customer security telemetry and on application
security logs, so an archive path is an obligation rather than an option. Which tables archive, for
how long, and to what — [`time-keyed-tables-with-bounded-retention.md`](time-keyed-tables-with-bounded-retention.md)
owns that. What this decision imposes is that whatever performs the archive holds the same bypass as
backup, per tenant database.

**No AUTOCOMMIT for control-plane queries.** `set_config(..., true)` is a silent no-op under
AUTOCOMMIT. Existing AUTOCOMMIT connections (`bootstrap/runner.py:170`,
`providers/container.py:103`) are provisioning paths and remain correct, but the pattern must not
spread to query paths.

---

## 6. Fixes folded into this work

- `create_alert_partition_if_not_exists` checks `SELECT 1 FROM pg_class WHERE relname = partition_name`
  with no `relnamespace` filter. Benign today, wrong the moment two schemas in one database both hold
  an `alerts_YYYY_MM`. Add `AND relnamespace = current_schema()::regnamespace`.
- The Redis sentinel-collision issue is **out of scope** but unaffected by this ADR and still open:
  `enrichment.py:321` and `tenant_rate_limiter.py:316` namespace on a tenant id that can be the
  sentinel.

---

## 7. Invariants to enforce in CI

These are the difference between this design holding and eroding. Each is a test, not a convention.

1. **Deployment-shape references are ratcheted.** No module outside `shared/tenant/` and
   `license_service.py` may reference deployment mode or `isinstance` a provider. Current baseline:
   40 references across 13 files; the count must only fall.
2. **`DeploymentProfile.default_tenant_id is None` whenever more than one tenant can exist.** Service
   refuses to start otherwise.
3. **Deployment shape is explicitly declared, with no default.** Absent or unparseable configuration
   is a startup failure, not a silent fall back to single-tenant. Assert no code path reads deployment
   shape with a default value — this is what makes invariant 2 reachable (§2.5).
4. **No `tenant_databases` row for any sentinel** under multi-tenant mode — the accidental protection
   of §1.2, made explicit.
5. **`public` holds zero tables** in the self-hosted database (post-migration and at startup). This
   catches DDL that ran without `search_path`, which would otherwise land in `public` and become
   visible to both planes. Replaces a lint, which cannot observe runtime connection state.
6. **Every control-plane table with a `tenant_id` has RLS enabled and forced.**
7. **Tenant context does not survive pool checkout.** Set the GUC, return the connection, check out a
   fresh session, assert `current_setting('app.tenant_id', true)` is empty.
8. **No AUTOCOMMIT engine is used for control-plane queries.**
9. **Backups capture every row with RLS live.**

---

## 8. Validation gates

No question below can invalidate the decision. The design-killers identified during analysis were
resolved by documentation and established practice, and the two Phase 1 checks in §8.1 — the only
ones that could change anything — fail into a fallback or an implementation bug, not into a different
design. So there is **no separate proof-of-concept**: a PoC de-risks a decision before committing to
it, and with nothing left that could force a different design it would be the build with a delete step
at the end. These are validated *in place*, in the order §9 sequences the work.

### 8.1 Technical — validated during implementation

**Answer before starting** — needs a `psql` session, not a build:

| Question | Test | If it fails |
|---|---|---|
| Does `cloudsqlsuperuser` hold `BYPASSRLS`, and can it be granted? | `SELECT rolname, rolsuper, rolbypassrls FROM pg_roles WHERE rolname IN ('postgres','cloudsqlsuperuser', current_user)` | Affects Cloud SQL *exports* only; managed backups are storage-level. Policy-encoded admin (§2.6) already avoids the dependency |

**Phase 1 — the vertical slice** (§9). These are the only two that could change anything, so they come
first, with ~4 tables migrated rather than 90:

| Question | Test | If it fails |
|---|---|---|
| Does `FORCE RLS` filter the owning role on CNPG and Cloud SQL? | Enable, query as owner, assert filtered. Becomes invariant 6 (§7) | Control-plane protection for four small tables falls back to §3.1. Does not affect the plane split |
| Does tenant context survive pool checkout? | Invariant 7 (§7) | The fix is `set_config(..., true)`, already the decided design (§2.6) — so a failure means an implementation bug, not a design change |

**Later phases — measurements and cost validation, not gates:**

| Question | When | Consequence |
|---|---|---|
| Two-lineage runner with per-schema `schema_migrations` | Phase 2 | Effort estimate only |
| Provisioning time: collapsed base schema vs 92 migrations | After base schema exists | Datapoint; informs §2.11's claim |
| `pg_dump`/restore round-trip across both schemas **with RLS live** | Before first production deploy | Backup role constraint (§5) needs revisiting |

Three questions were retired along the way: expressing three audit query modes as policies and
measuring RLS overhead on a high-volume `audit_logs` (both moot once §2.7 moved tenant audit to the
tenant plane), and verifying CNPG `Pooler` behaviour — deferred to §5, since no pooler is deployed and
none is proposed.

---

## 9. Build order and follow-ups

**Build risky-first.** There is no separate PoC (§8); instead the order is chosen so the parts that
could still surprise us land while the blast radius of being wrong is a day's work rather than the
whole schema.

**Phase 1 — vertical slice.** Two schemas, two roles with role-level `search_path`, **one**
control-plane table under `FORCE ROW LEVEL SECURITY`, **one** tenant-plane table, and the session
dependency issuing `set_config(..., true)`. End to end, with invariants 6 and 7 as tests. This is the
whole of what §8.1 calls a gate, exercised against ~4 tables instead of 90 — so a surprise costs a
day, not a rewrite. The work is kept, not discarded.

**Phase 2 — two-lineage migration runner**, with the fleet semantics of §2.13, the split Helm hooks,
and the Compose bootstrap chain of §2.12. Local dev lands here rather than last because every later
phase is developed against it.

Plane assignment (§2.9) follows Phase 1 and is the decision most expensive to reverse, so it waits on
the gates rather than leading. What comes after it — install flow, provisioning tenant #1, wizard
validation — is product sequencing rather than this decision's to set.

This ADR moves from Proposed to Accepted once Phase 1 confirms the two gates.

**Follow-up, separate concern — audit durability and off-box export.** Out of scope here, but surfaced
by §2.7: audit currently lives entirely inside the operator's delete authority. The append-only trigger
exempts superusers, and `retention/postgres.rs` purges `audit_logs` at 365 days as `postgres`. Against
a privileged insider that is not a durable record, and no second database fixes it. The mitigation is
an append-only sink outside that authority — GCS with retention lock, Cloud Logging with a retention
policy, or forwarding to a customer-held SIEM (the strongest option for the "who from the vendor touched
my data" case, since the customer holds the copy).

A shell exists and is unwired: `analysis/siem/` ships a forwarder, connectors, and formatters, none of
which reference audit — it carries alerts only. Whether that shell comes across at all is an open
decision, so the follow-up cannot assume it.

Two compliance obligations make this a dated commitment rather than a hardening idea:
`AgileFramework/docs/compliance/logging-monitoring-and-audit-policy.md` requires log integrity
protections against tampering, and the retention policy puts a one-year archive on application
security logs. The arrival condition is the first attestation that asserts a tamper-evident audit
trail — ISO 27001 or SOC 2, whichever lands first. Wiring audit into a forwarder, plus a WORM
retention target, is still its own ADR and does not block this one.

**Reversal condition.** If first revenue turns out to be appliance sales rather than SaaS, the
cross-tenant security case largely evaporates — single-tenant deployments have no cross-tenant risk by
construction — and what remains is operator simplicity plus tidiness. That would justify deferring
everything except items 1 and 2 above.

---

## 10. References

**Internal**

- `EagerBeaver/src/shared/tenant/` — `dependencies.py`, `connection_manager.py`, `providers/base.py`,
  `providers/container.py`, `bootstrap/runner.py`, `middleware.py`
- `EagerBeaver/tests/unit/tenant/test_bootstrap_sql_parity.py`
- `database/migrations/` — `032`, `033`, `037`, `069`, `085`
- `docs/investigations/2026-07-31-deployment-shapes-audit.md`
- AgileFramework: `docs/architecture-m3-multi-tenant.md`, `docs/architecture-m3-vm-appliance.md`,
  `docs/stories/story-m3-1.4.md`, `docs/known-issues-archive.md` (ISSUE-203/207/209/210/213/215/219)

**External**

- [PostgreSQL — Row Security Policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [Cloud SQL for PostgreSQL — Data privacy strategies](https://docs.cloud.google.com/sql/docs/postgres/data-privacy-strategies)
- [Cloud SQL for PostgreSQL — Managed Connection Pooling](https://docs.cloud.google.com/sql/docs/postgres/managed-connection-pooling)
- [CloudNativePG — Connection pooling](https://cloudnative-pg.io/documentation/1.24/connection_pooling/)
- [CloudNativePG — Database management](https://cloudnative-pg.io/docs/devel/declarative_database_management/)
- [AWS — SaaS partitioning models (silo/bridge/pool)](https://docs.aws.amazon.com/whitepapers/latest/multi-tenant-saas-storage-strategies/saas-partitioning-models.html)
- [AWS Well-Architected SaaS Lens](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/saas-lens.html) — tenant isolation mindset (§2.6), tenant-aware operations (§2.7)
- [AWS — SaaS architecture fundamentals: control plane vs. application plane](https://docs.aws.amazon.com/whitepapers/latest/saas-architecture-fundamentals/control-plane-vs.-application-plane.html)
- [Azure Architecture Center — Control planes in multitenant solutions](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/control-planes)
- [Citus 12 — Schema-based sharding for PostgreSQL](https://www.citusdata.com/blog/2023/07/18/citus-12-schema-based-sharding-for-postgres/)
- [PostgREST RLS with transaction-scoped settings — Scaleway](https://www.scaleway.com/en/docs/serverless-sql-databases/api-cli/postgrest-row-level-security/)
- [OWASP — Multi Tenant Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Multi_Tenant_Security_Cheat_Sheet.html)
- [PgBouncer changelog](https://www.pgbouncer.org/changelog.html) (CVE-2025-12819, fixed 1.25.1)
