---
status: "proposed"
date: 2026-08-06
decision-makers:
consulted:
informed:
---

# Coarse-grained modular monolith with CI-enforced boundaries

## 1. Context and Problem Statement

### 1.1 What must be settled, and why it gates the backlog

The rebuild needs an architecture style chosen deliberately, because it gates most of the decision
backlog. Under a single deployable unit the seams left by the services not being carried across
become in-process module contracts; under anything else they are network boundaries with their own
failure modes, and writing them first means writing them twice.

### 1.2 Three capabilities are deferred, not abandoned

Only the endpoint-management capability is being brought across initially. `analysis`, `responder`
and `forwardingrelay` are dropped, but all three are expected back — alert ingestion and
enrichment, automated response, and relay tunnelling are product capabilities being deferred, not
abandoned.

**The style therefore has to make re-adding a substantial capability a routine operation, and must
not be a decision that is cheap now and expensive to reverse when the second capability arrives.**
An architecture that only works at one module is as unsuitable as one that only pays off at five.

### 1.3 The baseline is a distributed monolith

The proof-of-concept presents as microservices — five deployable units, nine Helm subcharts,
per-service ports, and a readiness report naming the pattern outright
(`AgileFramework/docs/implementation-readiness-report-2025-12-14.md:206`). None of the properties
that would make the decomposition load-bearing are present, and several actively couple the units.

It is a monolith that has been distributed, and it is the baseline any alternative is measured
against — including the reason "modular monolith" cannot be adopted on the strength of the name
alone. §1.4–1.8 set out the evidence; §1.9 records what holds.

### 1.4 The decomposition is nominal

**One unit holds the system.** `endpointmgr` carries 82% of the HTTP surface, at 278 route
decorators across 42 mounted routers:

| Unit | Python files | LOC | Route decorators |
|---|---|---|---|
| `endpointmgr` | 99 | 44,201 | 278 |
| `shared` | 90 | 35,047 | — |
| `analysis` | 48 | 16,572 | 37 |
| `responder` | 24 | 8,103 | 11 |
| `forwardingrelay` | 15 | 3,193 | 4 |

`shared` is the second-largest component and is not a service. Alongside these the Next.js
dashboard is 104,297 lines. The three smaller services are satellites of the first, not peers.

**The dominant unit is itself unpartitioned.** `endpointmgr` spans roughly fifteen domains — agent
lifecycle, licensing enforcement, tenant provisioning and deprovisioning, installer and packaging,
updates and rollouts, detections, forwarding, the response queue, TLS administration, identity
integrations, audit, settings, uninstall, usage metering, terms acceptance — as 36 API modules and
36 service modules with no grouping between them, one of which (`api/agent_registration.py`) is
2,656 lines. Splitting a system into services says nothing about whether the parts have internal
structure.

**The boundaries follow pipeline stages, not bounded contexts.** Ingest → analyse → respond means
all four services operate on the same three entities, `agents`, `alerts` and `commands`. Cutting a
pipeline into processes runs the cut lines through the data every stage shares, which is the
mechanism that produced §1.6. A split along bounded contexts would have given each unit tables it
owned.

### 1.5 Nothing enforces the boundaries

**The units are separated by directory names only.** There is no `pyproject.toml`, no `setup.py`,
and no per-service package: the five packages are siblings in one flat namespace, assembled at
runtime by four independent `sys.path.insert` calls, one per entrypoint
(`EagerBeaver/src/{analysis,responder,endpointmgr,forwardingrelay}/main.py`). `pyrightconfig.json`
mirrors this with a single `extraPaths` root. There is no import-linter, no lint configuration, and
`.pre-commit-config.yaml` carries only secret-scanning and file hygiene.

**Independent deployability, the definitional property, is absent.** One pipeline builds all seven
images from one commit and deploys the chart at one tag: `--set-string global.imageTag=${SHORT_SHA}`
(`cloudbuild-staging.yaml:322`, images at `:492-504`). There is no per-service version, no
per-service pipeline, and no contract test between any two units.

**Four dependency edges cross the service boundaries**, in ten import statements across six files:

| Edge | Imported | Where |
|---|---|---|
| `responder` → `endpointmgr` | `forwarding_service`, `license_service`, `command_executor` | `responder/tools/redirect_traffic.py:33`, `responder/services/integration_service.py:61`, `responder/services/langgraph_agent.py:1965` |
| `responder` → `analysis` | `enrichment_service` | `responder/automation/actions/search_ioc.py:163` |
| `endpointmgr` → `analysis` | SIEM config and forwarder | `endpointmgr/api/siem_settings.py:26,33` |
| `shared` → `endpointmgr` | `license_service`, ×4 | `shared/middleware/license_gate.py:118,191,257,365` |

The last is a dependency inversion: the common library reaches into a leaf service, so every
service that mounts the licence gate acquires `endpointmgr`. The build pays for these in full — the
responder image copies all 44,201 lines of `endpointmgr` to satisfy three imports, and the
endpointmgr image copies `analysis/siem` plus a stub package marker
(`EagerBeaver/Dockerfile.responder:45`, `Dockerfile.endpointmgr:45`).

### 1.6 Data, build and configuration are shared across the units

**The data model is global.** All 48 ORM tables live in `shared/db_models.py` (39 of them),
`shared/tenant/db_models.py` and `shared/forwarding_models.py`; no service owns a table, and none
has a schema or database role of its own. `alerts` is written by `endpointmgr` and then enriched in
place by `analysis`. One lineage of 92 migrations serves every unit through a shared
migration job, and every unit receives the same database, role, Redis logical database and Neo4j
instance. **Shared mutable data is what makes independent deployment impossible irrespective of
process count, and it is the property the service split never had.**

**Cross-unit workflow is carried in the database.** `alert_processor.py:335` writes alerts with
`is_enriched=False` expecting a different unit to enrich the row and flip the flag, and `:312`
resets it on a dedup merge to re-trigger — a distributed workflow with no orchestrator, no timeout
and no compensation. There is no message bus; Redis pub/sub exists on exactly one surface, the
relay's session and command channels.

**Build and configuration are single-tenanted across the units.** One `src/requirements.in`
compiles to one hash-pinned lockfile installed into all four images, so LangGraph, Vertex AI and
the Neo4j driver ship inside the 3,193-line relay; a dependency bump is a four-service rebuild and
a four-service risk assessment. One 105,667-line test suite with one `conftest.py`. One `.env`.

**Change locality is absent.** `shared/` is touched by 48 of the last 300 commits, and each of
those rebuilds and redeploys all four units. Direct source coupling between units is low by
contrast — 29 of 300 commits touch more than one unit's source tree — which is the asymmetry that
makes consolidation cheap: the units are nearly separable in code and wholly inseparable in
deployment.

**Routing knowledge is duplicated, and the copies disagree.** The Kubernetes ingress routes five
path prefixes to five backend services (`liarbird-helm/templates/ingress.yaml:106-146`), while the
dashboard independently proxies to four of them through a `rewrites()` table and a catch-all
handler at `src/app/api/v1/endpointmgr/[...path]/route.ts`
(`EagerBeaver/src/dashboard/next.config.js:81-146`). `/api/v1/agent` reaches `endpointmgr`'s agent
port through the ingress and the *responder* through the dashboard. The `serverRuntimeConfig` block
that appears to configure this is dead — nothing reads it.

### 1.7 Scheduled work rides the request path, and replication is already configured

`endpointmgr` starts seven background workers in its lifespan — cleanup, uninstall expiry, agent
reaper, pending-response expiry, token expiry, session history, deprovisioning
(`endpointmgr/main.py:339-473`) — and `analysis` runs a continuous-processing task plus the alert
partition roller (`analysis/main.py:150,214`). The only advisory lock in the codebase guards tenant
bootstrap (`shared/tenant/bootstrap/migration_runner.py:345`); there is no leader election.

The production overlay nonetheless enables horizontal scaling: `endpointmgr` at `minReplicas: 2`,
`analysis` 2–10, `responder` 1–5, `dashboard` 2–5, all with `autoscaling.enabled: true`
(`values-gke-prod.yaml:174-268`). **So the reaper and the expiry sweeps are configured to run in
duplicate.** Production currently carries no tenants, which is why this is armed rather than
firing.

Two conclusions follow. Scheduled work must be a separate process role from request handling, and
that is a defect to correct rather than a capability to add. And per-unit resource shaping is the
one benefit the split genuinely delivers — though against `endpointmgr` rather than the dashboard,
the divergence is 2× memory at identical CPU (`responder` 2Gi/4Gi against 1Gi/2Gi), which is a
weaker case for separate units than the raw configuration suggests.

### 1.8 Failure isolation is partial, and locally inverted

Where a unit boundary sits on an agent-facing critical path it fails hard rather than degrading:
`api/agent_registration.py:2637-2648` returns **503 to the agent** when the forward to `analysis`
fails, and the Postgres write has already happened at `:2521-2585`, so agent retries re-submit work
that already landed.

Deployment-level fate-sharing is stronger than the split suggests. `nginx/nginx.conf` resolves
upstream names at config load, so an unresolvable `analysis_backend` (`:129`), `responder_backend`
(`:134`) or `forwardingrelay_backend` (`:145-146`) takes down the entire ingress, `endpointmgr` and
the dashboard included.

### 1.9 What holds

**Internal structure is better than the topology.** `shared/tenant/` has a working provider seam
that already absorbs both deployment shapes (`providers/`, `connection_manager.py`,
`dependencies.py`); `shared/repositories/` is a clean data-access layer; `shared/middleware/`
centralises JWT, RBAC and rate limiting.

The layering is only partly applied, though — routers issue 81 ORM `select()` calls directly and 6
of 37 API modules use a repository — which is §1.5's finding in a different register: **structure
that is available but not required does not hold.**

**One capability boundary in the estate does work, and it is the reference.** AccountManagement is
a separately deployed peer with its own repository, its own Prisma schema of 26 models
(`Customer`, `Subscription`, `Entitlement`, `License`, `PricingPlan`, `UsageRecord`, …), its own
release cadence, and an explicit contract: `endpointmgr` fetches
`GET /api/v1/tenants/{tenantId}/entitlements` over mTLS with a cache and a restrictive default on
failure (`endpointmgr/services/entitlement_client.py:183,196`).

Two things follow. Separation succeeds in this estate when the boundary follows a capability that
owns its data — so the case against splitting here is about *these* boundaries, not about splitting
in principle. And **licensing authority is already external**: what belongs to this system is a
consumer — a client, a cache and an enforcement gate — not the entitlement model.

### 1.10 What enforcement is available

Module-level data ownership cannot be enforced by the database in this estate. Grants are the
obvious mechanism and they are inert: the control-plane decision rejects role-based
`GRANT`/`REVOKE` on migration `069`'s finding that it is ineffective while the application role owns
the tables, which is the typical configuration in CNPG-managed and Cloud SQL-managed deployments
([`control-plane-tenant-plane-separation.md`](control-plane-tenant-plane-separation.md) §3.7).

Two mechanisms do survive owner topology, and neither reaches modules:

- **Connection**, for tenant isolation — a separate database per tenant behind a separate engine.
  Scoped to tenants, and modules within a plane share both engine and role.
- **`FORCE ROW LEVEL SECURITY`**, for the four tenant-scoped tables that stay in the control plane
  (§2.6 there). `FORCE` is the documented answer to owner-bypass on both Postgres and Cloud SQL, so
  this is genuine database enforcement — but a row policy filters *rows by predicate*, and module
  ownership is a claim about *which code may reference a table*. No policy can express it.

The plane boundary itself is convention plus a CI assertion rather than grant-enforced (that ADR's
§4, Negative), which is the same conclusion arrived at one level up.

Schema-per-module remains available as organisation rather than enforcement: one role per plane
with every module schema on its `search_path` resolves unqualified queries correctly while table
names stay globally unique, so engine and pool counts are unchanged. That is a refinement to settle
alongside the storage topology, not here.

**So module boundaries are enforced in CI under every option in §3.** This removes data ownership
as a discriminator between them and leaves the comparison resting on code boundaries and
deployment cost.

### 1.11 What this decision has to settle

The choice is not monolith versus microservices; §1.3–1.8 show the two can be indistinguishable in
every respect that matters while differing in deployment cost. What distinguishes the options is
**what enforces the module boundaries, and what the cost is of adding the second, third and fourth
module** once §1.2's capabilities return.

Two facts constrain the answer less than they appear to. The agent wire contract fixes the
`/api/v1/endpointmgr/…` path namespace, but a path prefix is not a deployment topology — the
agent-facing plane is already a *listener* rather than a service, two uvicorn servers sharing one
app instance (`endpointmgr/main.py:870-900`). And the two deployment shapes are absorbed by the
tenant provider seam (§1.9) rather than by the service split, so neither shape argues for a
particular number of processes.

## 2. Decision Drivers

* **The decision principles in `CLAUDE.md`** — the simplest option that leaves a more complex one
  reachable; applied to the design rather than the guardrails; deferring a boundary only where
  moving it later stays local.
* **Re-adding a deferred capability must be routine** (§1.2) — a module, not a project.
* **Boundaries need a check that can fail.** §1.5 and §1.9 are the same finding twice: the
  proof-of-concept had a repository pattern, a shared layer and process boundaries, and drifted
  through all three because nothing tested any of them.
* **Boundary placement must not be guessed.** A boundary in the wrong place costs the seam *and*
  its relocation, and an enforced boundary is harder to move than an unenforced one.
* **Enforcement is a check on the author's future self, not a coordination device.** 1,292 of 1,390
  commits in the proof-of-concept come from one author, so the team-autonomy benefit of a split has
  never had a team to accrue to — and the drift in §1.5 happened anyway, without a second pair of
  eyes to appeal to.
* **One deployment mechanism for both shapes.** SaaS runs on GKE and self-hosted on a VM image with
  embedded K3s, both via Helm, so the topology must be one chart parameterised by values rather
  than two deployment stories.

## 3. Considered Options

* **A — Monolith, conventional structure.** One deployable unit; modules are folders; nothing
  checks dependencies or data ownership.
* **B — Coarse-grained modular monolith.** One deployable unit; a small number of modules with
  CI-enforced import contracts and per-module table ownership.
* **C — Modular monolith with separable modules.** B, plus every inter-module call through a facade
  that could become a network call.
* **D — Service per capability.** One unit per capability, each with its own package, image and
  release, communicating over versioned contracts.
* **E — The proof-of-concept baseline.** Multiple units over a shared ORM, database and lockfile.

## 4. Decision Outcome

Chosen option: **"B — Coarse-grained modular monolith"**, because it is the least complex option
that satisfies §1.2, and because §1.10 removes the only advantage the more complex options had over
it. With data ownership enforced in CI under every option, D pays deployment, routing and network
cost for code boundaries no stronger than B's and data boundaries identical to B's. C pays an
immediate price — no cross-module transactions or joins at the seam — to preserve separate
deployability that one module might eventually want, when adding a facade at one seam later is a
module-local change. A is rejected on §1.4: `endpointmgr` is what a monolith with conventional
structure becomes, and it is the least navigable component in the estate.

**Coarse means around five modules, not fifteen.** The capability set is given by the product
rather than inferred, and the plane assignment is already named by the control-plane decision, so
this is encoding boundaries we have evidence for rather than guessing:

| Module | Plane | Notes |
|---|---|---|
| `endpoints` | Tenant | Agent lifecycle, registration, manifests, commands and responses, updates and rollouts, installer, detections, policies, settings, uninstall. **Deliberately unsplit** |
| `tenancy` | Control | Tenant registry, provisioning and deprovisioning, plane bootstrap |
| `entitlements` | Control | Consumer of AccountManagement (§1.9) — client, cache, enforcement gate. Thin by construction |
| `metering` | Control + tenant | Reads tenant-plane counters, writes control-plane usage records. The one known straddler |
| `identity` | Control | Platform users, sessions, registration tokens. Contents are decision 10's to settle |
| `platform` | — | Cross-cutting layer: authn/authz middleware, audit, telemetry, connection acquisition. **A layer, not a feature module** |

`endpoints` stays whole because its internal seams are the ones nobody can place yet. Splitting a
tenant-plane module later preserves the connection, so it is a directory move and a contract
update — the deferral the principles ask for. `ingestion`, `response` and `relay` join this list as
modules when their capabilities return.

Deployment is **two process roles from one image**, which is the axis the proof-of-concept
conflated with capability: `web` (both listeners, stateless, replicated, HPA on) and `scheduler`
(all background work, single replica or leader-elected). Three images total — backend, dashboard,
migration job — against six today.

### 4.1 Consequences

* Good, because §1.7's duplicate-scheduler hazard is resolved by construction: background work
  cannot run in a replicated process because it does not run in the `web` role at all.
* Good, because the routing contradiction in §1.6 is deleted rather than fixed — with one backend
  there is nothing for an ingress and a dashboard config to disagree about. Five ingress path
  rules become one.
* Good, because a returning capability costs no image, no Deployment, no ingress rule and no chart
  change.
* Good, because the four import edges in §1.5 become intra-process module contracts, and the three
  build-time workarounds — the whole-`endpointmgr` copy, the `analysis/siem` partial copy, the
  vendored SQL tree — have nothing left to work around.
* Good, because one lockfile stops being a coupling defect and becomes correct: one unit, one
  dependency set.
* Bad, because replication multiplies connection pools in a way the proof-of-concept never faced.
  The tenant engine LRU caches 20 engines at `pool_size=5` plus 10 overflow — roughly 100 steady
  and 300 burst connections *per process* — so replica count, `TENANT_ENGINE_POOL_SIZE` and
  `max_cached` become coupled and a pooler moves from optional to likely.
* Bad, because per-unit resource shaping is forfeited. §1.7 puts the real divergence at 2× memory
  at identical CPU, so the `web` role is sized for the most demanding module it contains.
* Bad, because the blast radius of a bad deploy is the whole backend. Mitigated by the fact that
  §1.5 shows it already is, and §1.8 that `nginx` makes it worse than the topology implies.
* Neutral, because the dashboard remains a separate deployable under every option — Next.js cannot
  run in the Python process. Its scope is decision 11's.

### 4.2 Confirmation

Each assertion is a fitness function — an automated, objective check on an architectural
characteristic, failing the build rather than relying on review. Items 1–5, 7 and 8 are CI tests;
item 6 is a CI configuration change.

1. **`platform` imports no feature module.** An import-linter contract, run in CI. This is the
   check that would have failed on `shared/middleware/license_gate.py:118` (§1.5).
2. **No feature module imports another feature module's internals.** An import-linter layered
   contract permitting imports only of a module's declared public surface (`<module>/api.py`).
3. **Every ORM table is declared by exactly one module.** A test walking the SQLAlchemy metadata
   and asserting each `__tablename__` resolves to one owning module, with module table sets
   disjoint. Fails on a table added to `platform` or to two modules.
4. **No module acquires its own database connection.** A test asserting that connection acquisition
   appears only in `platform`, and that no feature module references the global client. This is the
   invariant that keeps plane reassignment local (§1.10) and it is the one the proof-of-concept
   fails in 44 of ~62 database-touching files.
5. **The `web` role starts no background work.** A test asserting the web application's lifespan
   registers zero schedulers and creates no long-lived tasks. Fails on the §1.7 pattern.
6. **Marked integration tests run in CI.** In a single-unit shared-database design the datastore is
   the module contract, and it is currently the part CI does not exercise — `pytest -m "not
   integration and not neo4j and not slow"` at ~51% unit coverage. Requires Postgres and Redis
   service containers.
7. **Queries are constructed only in repository modules.** A test asserting that `select(`,
   `update(` and `delete(` appear nowhere outside a module's repository package. The repository
   layer being available but not required is what let 81 direct `select()` calls accumulate in
   `endpointmgr`'s API modules alone (§1.9).
8. **No source file sits outside a declared module root.** A test asserting every source file
   resolves under `platform` or one of the feature modules. It fails on code arriving as a flat
   tree, which is §1.4's failure mode.

Two things these do not reach. There is no check for cross-module joins at the SQL level; assertions
3 and 4 constrain where models and connections may be referenced, which is the enforceable surface. A
query joining two modules' tables from a module that owns both models' imports would pass, and is
left to review. And whether a module's *internal* grouping is sensible is not expressible as a check
— assertion 8 catches the flat tree, not a badly divided one. A file-length ceiling would have caught
`api/agent_registration.py` at 2,656 lines, but that is linter configuration rather than an
architectural characteristic.

### 4.3 Constraints Imposed

* **Connection acquisition goes through one chokepoint in `platform`.** This is what makes plane
  assignment and table-to-module assignment locally revisable; without it both become diffuse and
  the deferrals in §4 stop being cheap.
* **Scheduled work may not be started by the `web` role.** Any new periodic job is registered with
  the `scheduler` role, and needs a leader-election or advisory-lock story if `scheduler` is ever
  replicated.
* **Adding a module must not require an ingress change.** One API path namespace served by one
  Service; a module that cannot live under it is an upgrade-path decision, not a routing change.
* **Table names stay globally unique**, so that schema-per-module remains available as organisation
  later (§1.10) without schema-qualifying application queries.
* **Replica count is sized against pool capacity**, not independently. See §4.1.
* **Decisions 4–7 state the owning module and its public surface.** Those seams become module
  contracts under this decision, so each ADR names which module owns the capability and what
  `<module>/api.py` exposes, rather than describing a network boundary.
* **`endpoints` may be subdivided without an ADR; the plane assignment may not.** Subdividing a
  tenant-plane module preserves the connection. Moving a table between planes is a data migration
  and a change to §1.10's assumptions.

## 5. Pros and Cons of the Options

### 5.1 A — Monolith, conventional structure

* Good, because it defers boundary placement entirely, which is the one thing we are least equipped
  to decide now.
* Good, because it removes most of §1.5–1.8 for free: no `sys.path` assembly, no duplicated
  Dockerfiles, no cross-unit imports, no routing duplication.
* Neutral, because the tooling gap between A and B is small — import contracts are configuration,
  not architecture. A's advantage is deferring *placement*, not avoiding *cost*.
* Bad, because `endpointmgr` is this option at scale: 44,201 lines, fifteen domains, 36 API and 36
  service modules with no grouping, one file at 2,656 lines (§1.4).
* Bad, because the violations accrue in the meantime. A → B is cheap in tooling and expensive in
  cleanup, and the cleanup grows with every merged module.
* Bad, because §1.9 is direct evidence of the failure mode: available-but-unrequired structure
  eroded here under a single author, with no coordination pressure to blame.

### 5.2 B — Coarse-grained modular monolith

* Good, because re-adding a capability is a module rather than a project (§1.2).
* Good, because it encodes boundaries that already have evidence — the product's capability set and
  the control-plane decision's plane assignment — instead of guessing.
* Good, because enforcement is CI configuration plus five tests, which is the cheapest available
  answer to §1.9's erosion finding.
* Neutral, because it forfeits per-unit resource shaping, the one benefit §1.7 credits to the
  existing split, at a measured 2× memory and identical CPU.
* Bad, because it commits to a module decomposition now. Mitigated by coarseness: five modules,
  with subdivision left additive and explicitly permitted without an ADR (§4.3).
* Bad, because one bad deploy affects the whole backend.

### 5.3 C — Modular monolith with separable modules

* Good, because a module can be lifted into its own process without rewriting its callers.
* Bad, because it gives up cross-module transactions and joins at every seam, immediately, for a
  property one module might eventually need.
* Bad, because the deferral is cheap: B → C is a facade at one seam, module-local, and reachable
  when a trigger fires (§6).

### 5.4 D — Service per capability

* Good, because AccountManagement demonstrates the pattern working in this estate — own data, own
  cadence, explicit contract, defined degradation (§1.9). The objection is to these boundaries, not
  to the pattern.
* Good, because it delivers genuine failure and resource isolation, which §1.7 shows is the only
  benefit the current split earns.
* Bad, because §1.10 removes its principal advantage: per-service data ownership cannot be enforced
  by the database here, so its data boundaries are CI-enforced exactly as B's are.
* Bad, because it recreates the per-unit routing, pipeline, contract-test and dependency surface
  that §1.5–1.6 price, for boundaries that are nearly separable in code and would still share a
  datastore.
* Bad, because it is the most expensive option to reverse if the capability boundaries turn out
  wrong, and §1.4 shows the last attempt drew them along pipeline stages.

### 5.5 E — The proof-of-concept baseline

* Good, because it is running in production, and retiring capabilities is already a Helm toggle
  rather than a rewrite.
* Good, because per-service HPAs and resource profiles are wired (§1.7).
* Bad, because it is not the simple option under any reading: it carries B's structural cost plus
  five images, ten Dockerfiles, nine subcharts, five pipelines and the duplicated routing of §1.6.
* Bad, because every distribution cost is paid and, of the benefits, only resource shaping is
  returned (§1.7).
* Bad, because it is the corner. §1.5's inversions, §1.6's shared model and §1.8's inverted failure
  isolation are what this style produced here over 1,390 commits.

## 6. More Information

**Promotion trigger.** This ADR is Proposed. It promotes when a Phase 1 vertical slice runs with
§4.2's assertions 1–5, 7 and 8 green and assertion 6's service containers in CI. Decision 13 governs
whether that slice happens during the documentation phase.

**Upgrade paths and their triggers.** Each rejected option remains reachable, and naming the
trigger is what stops "later if we need to" from having no arrival condition:

| To | Change required | Trigger |
|---|---|---|
| C, one seam | A facade at that module's public surface | A module needs separate deployment |
| D, one module | Lift the module to its own image and Deployment | Divergent workload or state affinity. `relay` is pre-identified: its WebSocket registry is in-process state keyed by agent, so a command for agent X must reach the replica holding X's socket |
| D, generally | Per-capability data ownership | A capability acquires its own datastore, as AccountManagement has |

**Gating.** Decisions 4–7 depend on this one and are constrained by §4.3. Decision 12 (language and
framework baseline) is partly settled by §4.2: the enforcement surface assumes Python tooling that
can express import contracts. Decision 8 supplies §1.10 and is not otherwise a dependency; its §1.1
attributes the duplicated bootstrap SQL to `EagerBeaver/` being a separate Docker build context,
and §4.1 removes that cause.

**Prior art the terms come from.** Three of this decision's mechanisms have standard names, used
above rather than reinvented: §4.2's assertions are *fitness functions* (Ford, Parsons and Kua,
*Building Evolutionary Architectures*); §1.4's diagnosis is a split along pipeline stages where
bounded contexts were needed (Evans, *Domain-Driven Design*); and §4's two process roles are the
process formation of *twelve-factor* factor VIII, with §4.3's stateless-`web` constraint following
from factor VI. Factors I and II also name §1.5's and §1.6's findings — a shared library across four
apps, and dependencies neither declared nor isolated.

The cloud vendors' frameworks are organised by quality attribute rather than by decomposition style
and do not speak to this choice. Their multi-tenancy guidance does, and is already cited where it
applies:
[`control-plane-tenant-plane-separation.md`](control-plane-tenant-plane-separation.md) §2.1.

**Provenance.** §1.4–1.8's cost accounting is corroborated by an independent assessment against
proof-of-concept commit `14d9e9fb`, held at
[`../investigations/2026-08-05-architectural-style-assessment.md`](../investigations/2026-08-05-architectural-style-assessment.md).
It reaches the same classification from the same artifacts and carries the fuller cost ranking,
including four consequences this ADR does not reproduce: the inert SIEM control plane, the
`neo4j_password` dependency in agent JWT authentication, the `MULTI_TENANT_MODE` default, and the
partition roller's placement. Those bear on decisions 4, 6 and 9, and on the Neo4j exclusion, rather
than on this one. The
claims above cite primary artifacts so that this ADR does not depend on it.
