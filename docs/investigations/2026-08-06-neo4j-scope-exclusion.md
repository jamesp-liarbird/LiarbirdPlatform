# Neo4j — what the graph held and what depends on it

**Date:** 2026-08-06
**Scope:** every Neo4j surface in the LiarbirdServer — the graph data model, its writers and
readers, the two tenant-lifecycle steps, the endpointmgr admin API, the secrets contract, and the
deployment charts. Establishes what the graph was for and what changes in its absence.
**Repos examined:** LiarbirdServer (`14d9e9fb`, 2026-08-05), AgileFramework (`2fae214`, 2026-07-29 —
`ADR-MT-000`, `ADR-MT-007`, the M1 architecture archive, published compliance statements)
**Method:** establish the graph's purpose from its schema and node model rather than from prose;
establish dependency direction by locating every writer and every reader; and for each surface that
touches Neo4j, determine from the code whether absence degrades or fails. A dependency counts as
load-bearing only where something reads what was written.

## TL;DR

The graph is the **internal data model of `analysis` and `responder`** — alert enrichment and MITRE
mapping — not a platform-wide store. Every writer lives in one of those two services (§2). Both are
excluded by the endpointmgr-first constraint, so under that scope nothing populates the graph and
nothing reads it.

The two tenant-lifecycle couplings behave in opposite directions (§4). Deprovisioning's graph export
is **already soft**: it swallows every exception as a warning and emits the `graph_export/`
directory only when the graph returns data, so an absent, unreachable or empty Neo4j produces
exactly the archive an absent Neo4j would. Provisioning is **hard and ungated**: `STEP_NEO4J_DB`
raises on failure and is the one Neo4j path that never consults `NEO4J_ENABLED`, so tenant onboarding
requires a reachable graph database that nothing subsequently reads.

The same asymmetry appears in the secrets contract (§5): `neo4j_password` is a **required** field of
the bundle endpointmgr loads to authenticate agents, so agent JWT authentication cannot initialise
without a graph credential.

No external commitment names graph data (§7). The one published export promise is generic as to
format and enumerates no artifacts. `ADR-MT-007`'s `graph_export/nodes.json` and
`relationships.json` are an internal specification, and the code implementing them already degrades
to omitting them.

Deception topology — the one candidate for a graph-shaped product concept — was never built (§8). No
topology code exists and the schema has no zone tables.

## 1. What the graph held

`EagerBeaver/src/shared/graph_models.py` defines the node types: `AlertNode`, `EntityNode`,
`EnrichmentNode`, `DeviceNode`, `UserNode`, `ProcessNode`, `NetworkConnectionNode`, `DecoyNode`,
`ContextNode`, `ResponseNode`, `TargetProcessNode`, `NetworkTargetNode`, `MitreTechniqueNode`. The
relationship types are `Contains`, `EnrichedBy`, `OriginatedFrom`, `Spawned`, `UsesTechnique`.

`EagerBeaver/src/shared/neo4j_schema.py:58-72` corroborates this from the constraint set: uniqueness
on `Alert`, `Enrichment`, `Entity`, `Context`, `Response`, and on the MITRE label family
(`MitreTechnique`, `MitreTactic`, `MitreMitigation`, `MitreD3fendTechnique`, `MitreEngageActivity`),
plus a composite `(type, value)` on `Entity`.

That is an alert-correlation and framework-mapping model: one alert, its extracted entities, their
enrichments, and the MITRE techniques they map to. `DecoyNode` is a participant in that graph, not a
deception inventory.

## 2. Every writer is in a service being dropped

| Writer | Service |
|---|---|
| `analysis/alert_storage.py` | `analysis` |
| `analysis/neo4j_processor.py` | `analysis` |
| `analysis/mitre_startup.py` | `analysis` |
| `responder/services/neo4j_agent_service.py` | `responder` |

`shared/neo4j_ingestion.py` is the library these call, not an independent writer. No module under
`EagerBeaver/src/endpointmgr/` writes to the graph.

## 3. Readers

Graph reads reach it through `shared/mitre_framework_service.py`, whose only importers are
`analysis/mitre_startup.py`, `responder/services/langgraph_agent.py` and
`responder/services/neo4j_agent_service.py`. The MITRE surface therefore leaves with the same two
services.

Endpointmgr's own Neo4j references are administrative rather than data reads (§6).

## 4. The tenant lifecycle

Neo4j appears in five endpointmgr files: `main.py`, `api/tenant_provisioning.py`,
`api/tenant_neo4j.py`, `services/tenant_provisioning.py`, `services/tenant_deprovisioning.py`.

**Provisioning fails hard.** `services/tenant_provisioning.py:125` declares
`STEP_NEO4J_DB = "neo4j_db"`; step 3 (`:261-278`) calls `neo4j_manager.provision_database(slug)`
under `asyncio.timeout(STEP_TIMEOUT_NEO4J)` and raises `ProvisioningError` on a falsy return
(`code="NEO4J_PROVISION_FAILED"`) or on timeout (`code="TIMEOUT"`). The rollback branch at `:612`
drops the database again with `export_first=False`.

`is_neo4j_enabled()` appears nowhere in `services/tenant_provisioning.py`,
`services/tenant_deprovisioning.py` or `shared/tenant/neo4j_manager.py`. The flag
(`shared/neo4j_config.py:77`, `NEO4J_ENABLED`, defaulting to `true`) gates the read and write paths
but not provisioning. Tenant creation therefore requires a reachable Neo4j regardless of the flag.

**Export degrades silently.** `services/tenant_deprovisioning.py:447` calls
`_export_neo4j_data` unconditionally, but the method itself (`:565-596`) wraps its whole body in
`except Exception` and logs a warning, and creates the `graph_export/` directory only inside
`if graph_data:`. An absent, unreachable or empty graph yields an archive with no `graph_export/`
entry and a completed export job.

So the export obligation `ADR-MT-007`
(`AgileFramework/docs/architecture-m3-multi-tenant.md:774`) specifies is, in code, best-effort.

## 5. The secrets contract

`shared/config/secrets.py` declares `neo4j_password: SecretStr = Field(..., …)` at `:95` — required,
not optional — mapped from `NEO4J_PASSWORD` at `:49` and listed as `"neo4j_password": True,  # Required`
at `:299`. It sits in the same bundle as `jwt_secret` (`:134`, `:375`).

Endpointmgr loads that bundle to authenticate agents, so agent JWT authentication cannot initialise
without a graph credential, in a service that performs no graph reads.

## 6. The endpointmgr admin surface

`api/tenant_neo4j.py` exposes three read-only routes, mounted at `main.py:835` under
`/api/v1/endpointmgr`:

- `GET /admin/neo4j/health` (`:67`)
- `GET /admin/neo4j/tenants` (`:119`)
- `GET /admin/neo4j/stats` (`:165`)

They are administrative, not agent-facing, so the agent wire contract does not constrain them. The
router-level comment at `:20-25` records that they expose cross-tenant topology — every tenant
database and its node counts — and that they were previously registered with no auth and no mode
check at all, reachable unauthenticated as a cross-tenant information leak. They now carry
`require_multi_tenant_mode` and `require_role(Role.ADMIN)` as router-level dependencies.

## 7. What was promised externally

The published commitment is generic:

> **Export** their data in a standard, machine-readable format (JSON or CSV) via the platform's data
> export function or by request to Liarbird support

— `AgileFramework/docs/compliance/information-security-and-privacy-governance.md:84`, repeated at
`docs/iso27001-policy-revisions.md:640`. It names a format and a mechanism, and enumerates no
artifacts. `graph_export/` appears only in `ADR-MT-007`'s internal specification.

## 8. Deception topology was never built

`AgileFramework/docs/archive/m1/architecture.md:486` (Decision 6.3, "Deception Topology as
First-Class Concept") specifies `GET /api/v1/topology/zones`, `/zones/{id}` and `/detections` with a
zone-oriented model, covering FR99–FR100. It is the one design in the estate that would have made a
graph store product-shaped.

No implementation exists. `EagerBeaver/src` contains no topology module — the only matches for
"topology" are unrelated proxy-chain comments in `shared/trusted_proxies.py` and the cross-tenant
comment in `api/tenant_neo4j.py` — and `database/init/01-schema.sql` has no zone or topology tables.

## 9. Deployment and cost surface

Beyond the Python client, Neo4j is carried by `liarbird-helm/charts/neo4j/` (Chart, values,
StatefulSet, Service), the global configmap and secrets templates, every `values-*.yaml` variant,
`appliance/charts/liarbird/templates/databases/neo4j-standalone.yaml` plus `values-ha.yaml`,
`appliance/release-manifest-staging.yaml`, `EagerBeaver/docker-compose.neo4j.yml`, and the
`docker-compose` and cloudbuild files.

`ADR-MT-000` (`AgileFramework/docs/architecture-m3-multi-tenant.md:382`) records the HA licensing
position: PostgreSQL via CloudNativePG and Redis via Sentinel are Apache 2.0, while Neo4j Causal
Clustering is **Enterprise (paid)**. It is the only paid database licence in the HA matrix.

The default test selection excludes graph tests — `"integration and not neo4j and not slow"` — so
the graph paths are not exercised in the standard run.

## 10. Surfaces, and what absence does to each

| Surface | Location | Behaviour without Neo4j |
|---|---|---|
| Graph data model | `shared/graph_models.py`, `shared/neo4j_schema.py` | Unreferenced |
| Ingestion | `shared/neo4j_ingestion.py` + 4 writers in `analysis`/`responder` | Leaves with those services |
| MITRE reads | `shared/mitre_framework_service.py` | Leaves with those services |
| Tenant provisioning | `services/tenant_provisioning.py:125`, `:261-278`, `:612` | **Fails** — raises, ungated by `NEO4J_ENABLED` |
| Tenant export | `services/tenant_deprovisioning.py:447`, `:565-596` | Degrades silently; omits `graph_export/` |
| Agent JWT auth | `shared/config/secrets.py:95`, `:299` | **Fails to initialise** — required secret |
| Admin API | `api/tenant_neo4j.py`, `main.py:835` | Three admin routes with nothing behind them |
| Charts and compose | `liarbird-helm/charts/neo4j/`, `appliance/charts/…`, `values-*.yaml` | Unreferenced infrastructure |
