# Architectural style — classification and cost accounting

**Date:** 2026-08-05
**Scope:** the architectural style of LiarbirdServer as deployed — the four EagerBeaver Python services, the shared layer, the data layer, and the deployment topology (Compose, Helm, cloudbuild). Places the repo in the context of the wider platform (AccountManagement, the Telemetryd agent, liarbirdctl) to distinguish repo-level style from platform-level style.
**Repos examined:** LiarbirdServer (`14d9e9fb`, 2026-08-05), AgileFramework (`2fae214`, 2026-07-29 — M1/M3 architecture documents and ADRs 001–005), AccountManagement (`840d996`, 2026-07-17 — the one adjacent service boundary), LiarbirdAgent (`3b69263`, 2026-07-27 — wire contract)
**Method:** classify against the properties that distinguish the candidate styles, not against intent. For each property — independent deployability, data ownership, failure isolation, dependency independence, change locality — establish it from artifacts that constrain runtime behaviour (image tags in pipelines, import graph, Dockerfile `COPY`s, DB model ownership, HTTP failure semantics) rather than from diagrams or service counts. A benefit is only credited when something in the repo exercises it.

> **Provenance.** Written as a LiarbirdServer working document and moved here to be version-controlled, because that repo's `docs/investigations/` is untracked. Kept verbatim as a dated record, so some of its references do not resolve as written: `ADR-006` and `ADR-007` are the two ADRs carried over as [`control-plane-tenant-plane-separation.md`](../adr/control-plane-tenant-plane-separation.md) and [`time-keyed-tables-with-bounded-retention.md`](../adr/time-keyed-tables-with-bounded-retention.md), and the sibling audits it cites are here as [`2026-08-05-endpointmgr-only-retirement-audit.md`](2026-08-05-endpointmgr-only-retirement-audit.md) and [`2026-07-31-deployment-shapes-audit.md`](2026-07-31-deployment-shapes-audit.md).

## TL;DR

LiarbirdServer is a **shared-database distributed monolith**: a modular-monolith-shaped codebase deployed as four services that cannot be deployed, versioned or evolved independently. Every cost of distribution is paid. Of the benefits that would justify it, independent deployment, independent data, independent dependencies, technology choice and team autonomy are all structurally unavailable; per-service scaling and resource shaping are the exception and are genuinely wired (§4).

The decisive artifact is `cloudbuild-staging.yaml:322` — `--set-string global.imageTag=${SHORT_SHA}`. All seven images are built from one commit and deployed at one tag, against one Postgres schema described by one 2,228-line ORM module that all four services import. Independent deployability is the definitional property of microservices and nothing in the repo exercises it.

The boundaries were drawn along **pipeline stages** (ingest → analyse → respond) rather than capabilities, so all four services operate on the same three entities — `agents`, `alerts`, `commands`. That is what produced the shared schema, and it is why the services are simultaneously highly coupled at the contract level and almost entirely uncoupled at the code level (§3): EndpointMgr has zero Python imports from Responder or the relay and exactly one from Analysis.

Three splits do earn their keep and are worth naming separately from the four that do not: the LLM path, the WebSocket relay, and the agent-facing TLS plane (§4). The consolidation already planned in `2026-08-05-endpointmgr-only-retirement-audit.md` is the right correction, with one risk that plan does not name: process boundaries were the only enforcement of separation present, and after consolidation **enforcement is CI-only**. No database-level backstop is available for module boundaries — ADR-006's plane split isolates tenants, not modules, and per-module grants cannot be added under the owner topology (§6).

## 1. Verdict by layer

Repo-level style and platform-level style differ, and conflating them makes the platform look worse than it is.

| Layer | Style | Basis |
|---|---|---|
| Liarbird platform (multi-repo) | **Small set of genuine services** | AccountManagement, the agent and liarbirdctl each have own repo, own release cadence, own datastore, and a versioned contract |
| **LiarbirdServer (this repo)** | **Shared-database distributed monolith** | §2 — independent deployability, data ownership, dependency independence and change locality all absent; failure isolation inverted where it matters most |
| EndpointMgr internally | **Monolith** | 44,201 LOC, 99 files, 38 API + 38 service modules, one FastAPI app on two ports; larger than the other three services combined |
| Data layer | **Shared schema, tenant-sharded** | 39 tables in one `shared/db_models.py`; sharded per tenant in SaaS, never per service |

Service sizes, for proportion:

| Service | Python files | LOC |
|---|---|---|
| endpointmgr | 99 | 44,201 |
| **shared** | 90 | 35,047 |
| analysis | 48 | 16,572 |
| responder | 24 | 8,103 |
| forwardingrelay | 15 | 3,193 |

`shared/` is the second-largest component in the repo and is not a service.

## 2. The classification, property by property

### 2.1 Independent deployability — absent

`cloudbuild-staging.yaml:322` deploys the whole chart at one image tag; the seven `:${SHORT_SHA}` images are listed at `:492-504`. There is no per-service version, no per-service pipeline, and no contract test between any two services. The nine `file://` subcharts in `liarbird-helm/Chart.yaml` carry independent `condition:` keys but a single chart version.

### 2.2 Data ownership — absent

One `src/shared/db_models.py` (2,228 lines, 39 `__tablename__` declarations) is imported at 68 sites across all four services. There is no per-service schema, no per-service database role, and no table that a service alone may write. `alerts` is written by EndpointMgr, enriched by Analysis and read by Responder.

Isolation within that schema is enforced two different ways with no principle distinguishing them, which ADR-006 §1.1 records in full: tables reached via `get_tenant_db()` are protected by the connection, tables reached via `get_platform_db()` by a `WHERE` clause. The filter class has already produced a cross-tenant defect (ISSUE-219).

### 2.3 Dependency independence — absent

All four Dockerfiles install the same 112-package `src/requirements.txt`. The 3,193-LOC forwarding relay ships LangChain, LangGraph, `google-cloud-aiplatform` and the Neo4j driver. A dependency bump is a four-service rebuild and a four-service risk assessment.

### 2.4 Failure isolation — partial, and inverted where it matters

The one place a service boundary sits on an agent-facing critical path fails hard rather than degrading: `api/agent_registration.py:2637-2648` raises **503 to the agent** when the forward to Analysis fails, and the Postgres write happens first (`:2521-2585`), so agent retries re-submit work that already landed.

Deployment-level fate-sharing is stronger than the service split suggests. `nginx/nginx.conf` resolves upstream names at config load, so an unresolvable `analysis_backend` (`:129`), `responder_backend` (`:134`) or `forwardingrelay_backend` (`:145-146`) takes down the entire ingress, EndpointMgr and dashboard included. Compose fails `up` outright on an unset `NEO4J_PASSWORD` at six sites.

### 2.5 Change locality — absent

`src/shared/` is touched by **48 of the last 300 commits**, and each of those rebuilds and redeploys all four services. The dependency also runs backwards: `shared/middleware/license_gate.py:118,191,257,365` imports `endpointmgr.services.license_service`, so the shared layer depends on a service.

Direct source-level coupling between services is low (29 of the last 300 commits touch more than one service's source tree), which is the asymmetry that makes consolidation cheap — see §3.

### 2.6 Communication — synchronous HTTP plus shared mutable state

There is no message bus. Redis pub/sub exists on exactly one surface, the relay's session and command channels (`endpointmgr/tasks/session_history_worker.py`, `endpointmgr/api/forwarding.py`, `forwardingrelay/services/{command_handler,session_manager}.py`).

Cross-service workflow is carried in the database instead. `alert_processor.py:335` writes alerts with `is_enriched=False` expecting a different service to enrich the row and flip the flag, and `:312` resets the flag on a dedup merge to re-trigger. This is a distributed workflow with no orchestrator, no timeout and no compensation.

### 2.7 Build-time workarounds for code that was split without untangling

| Artifact | What it does |
|---|---|
| `endpointmgr/api/siem_settings.py:26,33` | imports `analysis.siem`, resolved by `COPY src/analysis/siem` in **both** EndpointMgr Dockerfiles |
| `Dockerfile.responder:45` | copies the **entire** `src/endpointmgr` tree into the Responder image |
| `shared/tenant/bootstrap/sql/` | `database/init/` + `database/migrations/` vendored byte-identically, because the endpointmgr build context cannot see the canonical tree |

## 3. Why the boundaries landed here

The split follows the stages of one pipeline:

```
agent → EndpointMgr → Analysis → Responder → EndpointMgr → agent
        writes alert   enriches   decides     commands
```

Each stage operates on `agents`, `alerts` and `commands`. Cutting a pipeline into processes runs the cut lines through the data every stage shares, which is the mechanism that produced §2.2. A capability split — endpoint lifecycle, detection, deception, tenancy — would have given each service tables it owned.

Two contextual facts bear on judging the decision rather than the outcome:

- **The repo has had one primary author.** 1,292 of 1,390 commits (2025-09-13 → 2026-08-05). The benefit a service split mainly buys — independent teams shipping on independent cadences — has never had a team to accrue to.
- **Multi-tenancy was retrofitted at migration `032` of `092`** (ADR-006 §1.1). The product was single-tenant for the first third of its schema life, so `tenants` arrived as one more table in the existing lineage rather than as a plane, and tenant isolation had to be layered onto a schema that four services already shared.

The audit in `2026-08-05-endpointmgr-only-retirement-audit.md` states the resulting shape precisely: *"almost no code coupling and a lot of contract coupling."* Four services that near-separable in code, yet undeployable separately, are providing overhead without isolation.

## 4. What the split does buy

Credited only where something exercises it.

| Benefit | Verdict |
|---|---|
| **LLM fault isolation** | **Real.** Vertex AI/LangGraph is the least reliable dependency in the system; a Responder crash cannot take down agent ingest. If one service were extracted, this is the one |
| **WebSocket session isolation** | **Real.** Long-lived tunnel sessions have a different lifecycle, memory profile and restart cost from request/response APIs. This split survives the argument on technical grounds |
| **Agent-facing plane separately routable** | **Real.** EndpointMgr's 8004 sits behind nginx SNI passthrough, so agent traffic bypasses dashboard TLS termination — this is what preserves TOFU pinning |
| **Deployment-shape flexibility** | **Real, and proven.** Same code runs multi-tenant SaaS on GKE (Cloud SQL, Memorystore) and single-tenant on a K3s appliance; the datastore-swap toggles are exercised in `values-gke-saas.yaml:88,105,122` |
| **Retirement is a toggle, not a rewrite** | **Real.** `analysis.enabled`, `responder.enabled`, `forwardingrelay.enabled` and `neo4j.enabled` already exist in `liarbird-helm/Chart.yaml:38,43,58,74` |
| **Independent scaling and resource shaping** | **Wired, not yet load-tested.** `values-gke-prod.yaml` gives each service its own HPA with distinct bounds — analysis 2–10 (`:174-177`), responder 1–5 (`:191-194`), endpointmgr 2–10, dashboard 2–5 (`:264-267`) — and distinct resource profiles matched to workload, e.g. responder at 2Gi/4Gi for the LLM path against the dashboard's 512Mi/1Gi. Prod carries zero tenants, so the bounds are configured rather than exercised |
| Independent deployment | Absent (§2.1) |
| Independent data | Absent (§2.2) |
| Independent dependencies | Absent (§2.3) |
| Technology choice per service | Absent — four identical FastAPI/SQLAlchemy stacks on one lockfile |
| Team autonomy | No team to grant it to (§3) |

Internal design quality is better than the topology suggests, and is what makes consolidation viable: the repository pattern in `shared/repositories/`, the `TenantDatabaseProvider` seam with a settable provider, layered middleware, and per-service health and metrics endpoints.

## 5. Costs, ranked

| # | Cost | Why it ranks here |
|---|---|---|
| 5.1 | Isolation by `WHERE` clause on a shared schema | Has already produced a cross-tenant defect; the class stays open |
| 5.2 | Responsibility placed in the wrong service | Two live instances, one with a four-year fuse |
| 5.3 | Duplication as a standing tax | Already caused a seven-month outage |
| 5.4 | A feature whose control plane is inert because of the split | Silent — the UI reports success |
| 5.5 | Distribution overhead with no isolation returned | The defining cost; diffuse rather than acute |
| 5.6 | The monolith relocated rather than prevented | Determines whether consolidation succeeds |
| 5.7 | Test posture cannot see the coupling | Compounds 5.1 |

### 5.1 Isolation by filter, on a shared schema

ISSUE-219 — LangGraph resolved an alert from tenant A to tenant B's agent record on a hostname or UUID match — was found and fixed; the class remains, defended by review. Two multipliers, both from ADR-006 §1.2:

- `MULTI_TENANT_MODE` **defaults to `"false"`** (`middleware.py:46`) and is read at 40 sites across 13 files. One absent variable in one values file puts a service into single-tenant mode, stamping the default-tenant sentinel on every request it handles. The same variable also decides how strictly `responder/main.py:54-68` enforces `GCP_LOCATION`, so the same omission downgrades a data-residency check from fatal to a warning.
- Sentinel degradation **fails open on writes**: rows land under a tenant nobody owns, with no error, no constraint violation, and nothing anomalous in the audit trail.

ADR-006 addresses this directly and is the reason it is not ranked as an open design question here.

### 5.2 Responsibility placed in the wrong service

- **The alert-partition roller runs in Analysis's lifespan.** `analysis/main.py:40,150` is the only caller of `ensure_alert_partitions_for_next_n_months`, yet EndpointMgr writes `alerts`. Coverage is hardcoded through 2030-12. Retiring Analysis without relocating the call reintroduces ISSUE-212, a resolved P0, with a fuse no test can reach.
- **`neo4j_password` is a required secret that gates agent JWT auth.** `shared/config/secrets.py:299` marks it required; `get_secrets()` is what EndpointMgr calls for its JWT secret (`main.py:44,271`, `middleware/agent_auth.py:78-79`). Dropping Neo4j breaks agent authentication for a reason absent from the request path — and `tests/conftest.py:158` sets the variable suite-wide, so the test run cannot surface it.

### 5.3 Duplication as a standing tax

| Duplication | Consequence |
|---|---|
| Two Dockerfiles per service (`EagerBeaver/Dockerfile.<svc>` and `EagerBeaver/src/<svc>/Dockerfile`) built by different pipelines | Drifted; a missing `COPY` in the shipped set caused a seven-month outage (GH #390). The dashboard has a third that nothing builds |
| `database/` vendored into `shared/tenant/bootstrap/sql/` | Every migration is a manual two-place copy, caught after the fact by red CI |
| `init/01-schema.sql` (33,441 B) and `000_base_schema.sql` (28,861 B) | Two independently-maintained base schemas ~4.5 KB apart, reaching different deployments by different paths |
| ~10 deployment shapes, four live; ~8 stale Helm values overlays | Every deployment decision is made against a larger surface than exists |

### 5.4 A feature whose control plane is inert because of the split

SIEM forwarding runs its engine in Analysis and its configuration API in EndpointMgr, communicating through Redis keys. `get_siem_forwarder()` is a process-local singleton (`forwarder.py:760-768`) and EndpointMgr calls it with no Redis client at all four sites (`siem_settings.py:142,177,192,203`). In EndpointMgr's process `get_delivery_status()` returns zeros, `_is_circuit_open()` is always `False`, and `reset_circuit_breaker()` is a no-op. Config saved in the dashboard takes up to the 5-minute cache TTL to reach the engine.

### 5.5 Distribution overhead with no isolation returned

Five images, ten Dockerfiles, nine Helm subcharts, five cloudbuild pipelines, cross-service tracing and per-service secret wiring — against forfeited independent deploy, data, dependencies, technology choice and team autonomy (§4). This is the aggregate of §2 rather than a separate finding, and it is the reason the style has a name.

### 5.6 The monolith relocated rather than prevented

EndpointMgr is 44,201 LOC with `api/agent_registration.py` alone at 2,656 lines, and one FastAPI app object (`main.py:598`) serves both the agent fleet on 8004/TLS and the dashboard API on 8001 (`:875,879`). No lint or packaging boundary constrains its internal structure.

### 5.7 Test posture cannot see the coupling

CI runs `pytest -m "not integration and not neo4j and not slow"`, so the marked integration tests — the only ones exercising the shared datastore that *is* the inter-service contract — do not run. Unit coverage is ~51%. Mocking the datastore is what hid GH #342.

## 6. Target state

The consolidation in `2026-08-05-endpointmgr-only-retirement-audit.md` is the correct correction, and §4's accounting is why: four services with contract coupling and no code coupling return no isolation for their overhead. Two amendments.

**Separate the relay's product decision from its architecture decision.** Retiring `forwardingrelay` retires dynamic port-forwarding as a *capability*. On architecture alone the relay is the one split that survives scrutiny (§4) — WebSocket session lifecycle genuinely differs from request/response. If the capability is kept, the service should be too.

**Name the risk consolidation introduces.** Collapsing to EndpointMgr plus dashboard produces a single ~60k-LOC process, and process boundaries were the only enforcement of separation present. `shared/` already imports `endpointmgr` (§2.5) with nothing preventing it. Two mechanisms are available to make "modular" mean something once the boundaries stop being physical, and both are CI-level:

1. **Import-direction lint** — `shared/` may not import a feature module; feature modules may not import each other.
2. **Marked integration tests in CI** — in a single-process shared-DB design the datastore is the contract, and it is currently untested by CI (§5.7).

**No database-level mechanism is available for module boundaries**, and the reason is worth recording because the plane split looks like one. ADR-006 gives the *tenant* boundary genuine database-level teeth — a separate database per tenant reached through a separate engine, plus `FORCE ROW LEVEL SECURITY` on the four tenant-scoped control-plane tables — which is what closes the ISSUE-219 class. It does not extend to modules. Modules within a plane share one role and one connection, and per-module grants cannot be added on top: ADR-006 §3.7 rejects role-based `GRANT`/`REVOKE` outright on migration `069`'s verified finding that *"GRANT/REVOKE is ineffective when the application's DB role is also the table owner … the typical config in CNPG-managed and Cloud-SQL-managed deployments."* The plane boundary itself is therefore *convention plus assertion* rather than grant-enforced (ADR-006 §4 Negative), enforced by CI invariant 5 — `public` holds zero tables — rather than by the database.

Enforcement of internal structure after consolidation rests entirely on CI. That is a workable position, and it is the same conclusion ADR-006 reaches from the other direction in its §7: *"Each is a test, not a convention."* It is worth stating explicitly, because a design that assumes the database is backstopping module boundaries will not get a failure when the assumption breaks.

The resulting style, stated plainly:

> **A modular monolith** (EndpointMgr + dashboard), **a separately deployed control-plane peer** (AccountManagement — own repo, own Prisma schema of 26 models, own Cloud Run deploy, HTTP contract at `/api/v1/platform/tenants` plus mTLS entitlements with a Redis cache and a restrictive default on failure), **and a specialised long-connection service** if port-forwarding is kept.

AccountManagement is worth holding up as the reference: it is the one boundary in the platform with its own data, its own release cadence, an explicit contract, and a defined degradation path when the peer is unreachable.

## 7. Relationship to work already in flight

| Document | Relationship |
|---|---|
| ADR-006 (control plane / tenant plane) | Closes §5.1 — and its §3.7 and §7 are the basis for §6's finding that module boundaries have no database-level backstop available. Orthogonal to style: the plane split isolates tenants, not modules |
| ADR-007 (unpartitioned time-keyed tables) | Independent of style, but §5.2's partition roller is the coupling that makes the partitioning removable |
| `2026-08-05-endpointmgr-only-retirement-audit.md` | Corroborated in full. This document supplies the classification and cost accounting that justify its Stage 3, and adds the §6 enforcement gap |
| `2026-07-31-deployment-shapes-audit.md` | Supplies §5.3's shape count; the production surface is two shapes plus a dev stack |

## 8. Open items

| # | Item | Decision needed |
|---|---|---|
| 8.1 | Whether dynamic port-forwarding survives as a capability (§6) | Product — the architecture answer follows from it, not the reverse |
| 8.2 | Import-direction lint before or alongside consolidation (§6) | Engineering. Cheaper before: the violations are enumerable today and grow with each merged module. One of only two enforcement mechanisms available, so it is load-bearing rather than hygiene |
| 8.3 | Promoting marked integration tests into CI (§5.7) | Engineering. Requires Postgres/Redis service containers in `server-ci.yml`. The other of the two; the gap widens as the datastore becomes the sole contract |
| 8.4 | Whether `shared/` survives consolidation as a layer or dissolves into the modules it serves | Engineering. It is the second-largest component (§1) and its current inbound/outbound edges are the clearest measure of how modular the result will be |
