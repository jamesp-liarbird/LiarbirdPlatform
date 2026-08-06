# EndpointMgr-only retirement — coupling audit and plan

**Date:** 2026-08-05
**Scope:** retiring the `analysis`, `responder` and `forwardingrelay` services and the Neo4j dependency, so that LiarbirdServer ships as EndpointMgr (plus the dashboard) alone. Covers EagerBeaver Python services, the shared layer, the dashboard, Docker Compose, nginx, the Helm charts, the appliance, and the test suite.
**Repos examined:** LiarbirdServer (`14d9e9fb`, 2026-08-05), AgileFramework (`2fae214`, 2026-07-29 — schema provenance and incident history), LiarbirdAgent (`3b69263`, 2026-07-27 — alert wire format and build-mode gating)
**Method:** for each retirement target, three independent signals — (1) Python import graph into `endpointmgr`; (2) runtime contracts (HTTP calls, Redis channels, DB columns one service writes and another reads); (3) deployment wiring (Dockerfile `COPY`s, Compose `depends_on`, nginx upstreams, Helm conditions). A dependency is only called *hard* when its absence produces an error rather than a degraded path.

## TL;DR

There is almost no **code** coupling and a lot of **contract** coupling. EndpointMgr has zero Python imports from `responder` or `forwardingrelay`, and exactly one from `analysis` (`analysis.siem`, worked around by explicit Dockerfile `COPY`s). What binds the services is HTTP calls, Redis pub/sub channels, a required-secret declaration, and rows EndpointMgr writes expecting another service to finish them.

Four gotchas will bite before any feature work does, and three of them are silent: `neo4j_password` is a **required** secret consumed by agent JWT auth (§3.1); `POST /responses` returns **503 to the agent** when Analysis is unreachable (§3.2); nginx **refuses to start** on an unresolvable upstream (§3.3); and Compose `NEO4J_PASSWORD:?` guards fail `docker compose up` outright (§3.4).

Redis is **descoped** (§2.6) — its coupling to the retiring services is only the relay's pub/sub channels; everything else it does is EndpointMgr's own, including two controls that fail open. SIEM forwarding is **kept** and moves into EndpointMgr (§4).

MITRE tactics were never enrichment-derived — Analysis only relabelled a field — but the agent strips them to `None` in production builds by design, so keeping the column requires **deriving them server-side** from the detection profile (§6.1). One further silent trap: the rolling **alert-partition maintenance helper runs in Analysis's lifespan** and is its only caller, so retiring Analysis reintroduces a resolved P0 with a four-year fuse (§3.8).

## 1. Assumptions

These are the premises the plan rests on. Each is a decision, not a finding — revisit this list before acting on later sections.

| # | Assumption | If it changes |
|---|---|---|
| A1 | **Best-effort SIEM delivery is acceptable; durability is not required now.** | §4.4 becomes mandatory rather than advisory. The outbox seam is cheap now and a retrofit later; A1 is the single assumption most likely to flip. |
| A2 | **There are no existing customer SIEM integrations.** The EndpointMgr-native event shape is free to differ from `StandardEvent`. | §4.6 grows a compatibility-mapping task and a customer-notification step. |
| A3 | **No per-tenant Neo4j databases exist in any live deployment.** | §2.5 gains a data-disposition decision (export vs. drop) and a migration for recorded `completed_steps`. |
| A4 | **Redis stays.** It is not a retirement target. | §2.6 reopens: rate limiting, the entitlement stale-fallback and the revocation cache all need replacements, and two of them fail open silently. |
| A5 | **Alert enrichment (GeoIP, threat intel, rareness, graph ingestion) is being retired as a product capability**, not relocated. | §6 changes: `is_enriched` regains meaning and the alerts UI indicator stays. |
| A6 | **The dashboard stays in scope** and its broken surfaces get removed rather than left dead. | §5 becomes an accepted-breakage list instead of a work list. |
| A7 | **The attack-chain graph and Command Centre graph view are being retired.** | Neo4j cannot be removed; graph ingestion currently lives only in `analysis` (§2.3). |
| A8 | **Non-agent log sources are being retired** — Microsoft identity/Graph receivers, network receivers, and the `/api/integrations` catalogue. Agent telemetry becomes the only ingest path. | The parser pipeline and receiver managers must be relocated, which is most of `analysis`. |

## 2. Coupling map

### 2.1 Code-level: one import

`endpointmgr` has **no** Python imports from `responder` or `forwardingrelay`. The only cross-service import is:

- `EagerBeaver/src/endpointmgr/api/siem_settings.py:26` — `from analysis.siem.config import (...)`
- `EagerBeaver/src/endpointmgr/api/siem_settings.py:33` — `from analysis.siem.forwarder import get_siem_forwarder`

Both EndpointMgr Dockerfiles work around this explicitly:

```dockerfile
# SIEM settings API uses analysis.siem for config models and forwarder
COPY src/analysis/__init__.py /app/analysis/__init__.py
COPY src/analysis/siem /app/analysis/siem
```

(`EagerBeaver/Dockerfile.endpointmgr`, and again in `EagerBeaver/src/endpointmgr/Dockerfile`.)

`analysis.siem` is self-contained: stdlib, `aiohttp`, `cryptography`, `pydantic`, plus one reach into `shared.tenant.db_models` (`forwarder.py:246`). Relocating it is mechanical — 13 source files and 3 test files. Destination is `src/endpointmgr/siem/` (§4.2), not `shared/`, because EndpointMgr becomes the only consumer.

### 2.2 HTTP: EndpointMgr → Analysis, three call sites

| Call site | Target | Behaviour on failure |
|---|---|---|
| `services/alert_forwarder.py:30-32` | `POST http://{ANALYSIS_API_HOST}:{ANALYSIS_API_PORT}/logs`, batches of 50, 3 retries with exponential backoff | logs and returns `False` (`:208-209`) — **fail-soft** |
| `services/alert_processor.py:79-81` | `/api/v1/alerts/ingest` | Dead — the live path is the queue at `:352` drained by `_forward_alerts_task` (`:91-93`) into `AlertForwarder` |
| `api/agent_registration.py:2594-2596` | `POST .../logs` from the agent-facing `POST /responses` handler (`:305`) | **raises `HTTP 503` to the agent** (`:2637-2648`) |

The third is the only place a retiring service sits on an agent-facing critical path. See §3.2.

### 2.3 Semantic: `is_enriched` and the graph

`alert_processor.py` writes alerts with `is_enriched=False` (`:335`) and expects Analysis to enrich and flip the flag. On a dedup merge it resets the flag to `False` with the comment *"Re-trigger enrichment with merged data"* (`:312`).

Retiring Analysis permanently removes: GeoIP, VirusTotal/RST threat intel, rareness/anomaly scoring, the parser pipeline for every non-agent log source, and all Neo4j graph ingestion.

Graph ingestion is worth calling out precisely: `shared/graph_notifier.py` and `shared/neo4j_ingestion.py` are invoked **only** from `src/analysis/alert_storage.py:15,17,251,261` and `src/analysis/neo4j_processor.py:10,12,64,75,240`. Nothing in `endpointmgr` or `responder` writes the graph. **Retiring Analysis retires the graph whether or not Neo4j is kept.**

### 2.4 forwardingrelay: Redis channels and the agent manifest

Coupled through four surfaces, none of them imports:

- **Publishes** `fwd:commands:{tenant_id}` on command create/cancel — `api/forwarding.py:332-371`, called at `:426` and `:457`.
- **Subscribes** to `fwd:session_events:*` to persist session closes to `forwarding_session_history` — `tasks/session_history_worker.py:5,10,103-159`.
- **Scans** `fwd:sessions:*` hashes the relay writes, to list active sessions — `services/forwarding_session_service.py:64-73`.
- **Hands agents the relay URL** in their manifest — `api/manifests.py:839-842` injects `FORWARDING_RELAY_URL` (default `wss://app.liarbird.local/api/v1/tunnel`), proxied by `nginx/nginx.conf:223` to `forwardingrelay-service:8005` (`:145-146`).

Retiring the relay removes the dynamic port-forwarding feature entirely: 5 EndpointMgr files, `shared/forwarding_{models,queries,schemas}.py`, the `forwarding_*` tables, and the dashboard surfaces in §5.

A degraded agent path exists — `manifests.py:806` can blank `relay_url`, and `:846` notes the agent receives rules via heartbeat without needing the tunnel. See §3.6 for the residual risk.

### 2.5 Neo4j: four places, one of them invisible

1. **`shared/config/secrets.py:299` declares `neo4j_password` as required.** See §3.1 — this is the sharpest edge in the whole exercise.
2. **Tenant provisioning step 3** — `services/tenant_provisioning.py:260-277` creates a per-tenant Neo4j DB; failure is fatal and triggers rollback of the Postgres DB and tenant record (`:612-616`). The partial-tenant resume path enumerates `STEP_NEO4J_DB` in `completed_steps` (`:455-490`).
3. **Tenant deprovisioning** — graph export into the archive bundle (`services/tenant_deprovisioning.py:446-447,565-596`, fail-soft) and DB drop (`:876-888`, fail-soft, records `cleanup_status["neo4j"]`).
4. **Admin visibility** — `api/tenant_neo4j.py`, routed at `main.py:835`.

Under A3 all four are straight deletions with no data disposition and no migration for recorded steps.

**Stale comment to correct:** `docker-compose.yml` labels the Neo4j env block `# Neo4j (required for agent registration)`. `api/agent_registration.py` contains zero Neo4j references. Agent registration has never needed Neo4j.

### 2.6 Redis: descoped (A4)

Redis's coupling to the retiring services is **only** the four relay surfaces in §2.4. Everything else is EndpointMgr's own and stays:

| Surface | Files | Degradation without Redis |
|---|---|---|
| Rate limiting, incl. the SNI-passthrough enroll ceiling | `main.py:406-441`, `shared/middleware/rate_limiter.py`, `api/agent_registration.py:155-215` | **Fails open silently** — `rate_limiter.py:369` logs `"Redis unavailable, allowing request"` and allows |
| Agent revocation cache | `services/revocation_service.py:24,56-60,492-548`, `services/tenant_revocation_service.py` | Correct but slower — every `is_revoked()` hits Postgres |
| Entitlement cache + **stale fallback** | `services/entitlement_client.py:348-432`; no-TTL stale key at `:105,410` | Loses the fallback that survives an Account Management outage |
| Tenant key-count metrics | `main.py:445-448`, `shared/metrics/redis_metrics.py` | Metrics stop |
| Tenant key purge on deprovisioning | `services/tenant_deprovisioning.py:42` | Keys leak |

Two of these fail open, which is why removing Redis would *appear* to work while quietly removing protections. `redis_password` is already `Optional` in `secrets.py`, so no secrets change is needed.

**Work still required:** delete the four relay-only surfaces as dead code when the relay goes. `shared/redis_client.py` and `shared/tenant/redis_client.py` stay as-is.

## 3. Gotchas, ranked

| # | Gotcha | Failure mode |
|---|---|---|
| 3.1 | `neo4j_password` is a required secret | Silent until deploy; masked in tests |
| 3.2 | `POST /responses` 503s agents | Agent-facing, with duplicate-retry side effect |
| 3.3 | nginx won't start on a dead upstream | Ingress-wide outage, not partial |
| 3.4 | Compose `NEO4J_PASSWORD:?` guards | Loud — `up` fails immediately |
| 3.5 | Cold-cache silent drop in the SIEM forwarder | Silent event loss |
| 3.6 | Dead `relay_url` in enrolled agents | Unknown agent-side behaviour |
| 3.7 | `scheduled_cleanup` has no row claim | Pre-existing; matters if copied as a template |
| 3.8 | Alert partition roller lives in Analysis's lifespan | Silent, deferred to 2031 — reintroduces a resolved P0 |
| 3.9 | Appliance + cloudbuild scope | Under-scoping, not breakage |

### 3.1 `neo4j_password` is required, and agent auth consumes it

`shared/config/secrets.py` declares `neo4j_password` a **required** secret:

- `:49` — legacy env mapping `"neo4j_password": ["NEO4J_PASSWORD"]`
- `:95` — `neo4j_password: SecretStr = Field(..., ...)`
- `:299` — `"neo4j_password": True,     # Required`
- `:322-329` — missing required secrets raise `ValueError`

`get_secrets()` is what EndpointMgr calls to fetch its **JWT secret** — `main.py:44,271` and `middleware/agent_auth.py:78-79`. Removing Neo4j and dropping `LIARBIRD_NEO4J_PASSWORD` / `NEO4J_PASSWORD` therefore breaks agent JWT verification, for a reason that appears nowhere in the request path.

`EagerBeaver/tests/conftest.py:158` sets `LIARBIRD_NEO4J_PASSWORD` for the whole suite, so the test run will not catch this. Making the field optional is the first change in the sequence (§8).

### 3.2 `POST /responses` returns 503 to the agent

`api/agent_registration.py:2637-2648` raises `HTTP_503_SERVICE_UNAVAILABLE` when the forward to Analysis fails or times out. Two consequences:

1. Every agent response-batch submission fails once Analysis is gone.
2. The Postgres write happens **before** the forward (`:2521-2585`), so agent retries re-submit work that already landed.

The fix is to make the forward fail-soft, mirroring `AlertForwarder`'s behaviour, before Analysis is removed.

### 3.3 nginx refuses to start on an unresolvable upstream

`nginx/nginx.conf` declares `analysis_backend` (`:129`), `responder_backend` (`:134`) and `forwardingrelay_backend` (`:145-146`), with locations at `:191`, `:198` and `:223`. nginx resolves upstream server names at config load and exits with `host not found in upstream`. Removing a service before its upstream block takes down the whole ingress — EndpointMgr and the dashboard included. Upstream and location removal belongs in the same commit as each service.

### 3.4 Compose `NEO4J_PASSWORD:?` guards

`docker-compose.yml` fails `up` outright if the variable is unset, at six sites: `:140` (neo4j), `:197` (analysis), `:270` (responder), `:342` and `:357` (endpointmgr), `:451` (forwardingrelay). EndpointMgr's `depends_on` block additionally gates on `redis: service_healthy`, `neo4j: service_healthy` and `analysis-service: service_started`.

### 3.5 Cold-cache silent drop (inherited defect — see §4.3)

### 3.6 Dead `relay_url` in already-enrolled agents

Agents hold a pinned config version whose `relay_url` stops resolving. The server-side degraded path exists (§2.4), but the agent's behaviour against a dead `wss://` endpoint — in particular whether it retries in a tight loop — lives in the agent repo and is unverified here. Confirm before switching the relay off in a fleet with forwarding enabled.

### 3.7 `scheduled_cleanup` does not claim rows

`tasks/scheduled_cleanup.py:111-112` selects `status == "pending"` with no `SKIP LOCKED` and no atomic claim, so two replicas can pick up the same row. This is pre-existing and out of scope, but it is the closest structural template for the SIEM outbox (§4.5) — copy its shape, take the claim from `expire_stale_pending_responses`.

### 3.8 The alert partition roller lives in Analysis's lifespan

`alerts` is RANGE-partitioned by month on `timestamp` (`database/init/01-schema.sql:300`). Partition coverage is hardcoded through 2030-12 — 84 partitions from migration `070_extend_alert_partitions.sql` plus the matching DO block in `01-schema.sql`. Rolling that window forward is the job of `ensure_alert_partitions_for_next_n_months`, and its **only** caller in the repo is Analysis:

- `EagerBeaver/src/analysis/main.py:40` — import
- `EagerBeaver/src/analysis/main.py:150` — `await ensure_alert_partitions_for_next_n_months(session, n=6)`

`shared/partition_management.py:9` documents this placement explicitly. Retiring Analysis removes the self-healing mechanism; after 2030-12 alert INSERTs fail with `no partition of relation "alerts" found for row`.

This exact failure was ISSUE-212 (AgileFramework `docs/known-issues-archive.md:5978-5992`), a P0 resolved 2026-05-02 by adding the roller. Retiring Analysis without relocating it reintroduces the bug with a four-year fuse, invisible to every test.

**Action:** move the call into EndpointMgr's lifespan in Stage 1. EndpointMgr is the writer of `alerts`; Analysis only enriched them, so this is where the call belongs regardless of the retirement. Keep the existing failure semantics — log at WARNING, do not block startup.

**Related partitioning history**, useful when touching `alerts` DDL:

- ISSUE-280 (`known-issues-archive.md:1628-1632`) — liarbirdctl's retention `DELETE` filtered `alerts` on `created_at` rather than the `timestamp` partition key, defeating partition pruning.
- `story-m3-1.3.md:482-509,582-589` — GKE migrations failed `"alerts is not partitioned"` because SQLAlchemy `create_all()` produced non-partitioned tables before migrations ran; migrations were disabled in staging as a workaround.
- `story-m3-10.1.md:90,184,230` — the `alert_id` FK was removed from the ORM because partitioned tables cannot be FK targets.

No design rationale for the partitioning is recorded anywhere. It arrived with the initial schema as part of the `data/alerts/*.json` → Postgres migration, justified only as *"Partitioned by month for performance as data grows"* (`01-schema.sql:209-210`), with no volume estimate or retention requirement. The same programme declined to partition `audit_logs` (`story-m2-6.6.md:514`), so this was a judgement call rather than a policy.

### 3.9 Appliance and build pipelines

`appliance/liarbirdctl` (Rust) enumerates the five services across `src/retention/neo4j.rs`, `src/api/handlers/health.rs`, `src/k3s/health.rs`, `src/reset/factory.rs`, `src/update/sequence.rs`, `src/config/helm.rs`, `src/config/validation.rs` and `src/wizard/state.rs`. `appliance/charts/liarbird/values.yaml` carries per-service blocks at `:114,156,282,362,387`. Five cloudbuild pipelines build or promote all five images. This is a comparably-sized second workstream, easy to under-scope.

## 4. SIEM forwarding — the capability being kept

### 4.1 Where it lives today

The engine is in `analysis`; the configuration UI and API are in `endpointmgr`. They communicate through Redis keys.

- **Only producer:** `src/analysis/main.py:630-641`, specifically `:637-638` — `asyncio.create_task(_siem_forwarder.forward(str(tenant_id), event_dict))`. EndpointMgr never forwards.
- **EndpointMgr's surface:** `api/siem_settings.py` — config CRUD, connectivity test (`:177`), delivery status (`:192`), circuit-breaker reset (`:203`).
- **Shared state:** Analysis writes `tenant:{id}:siem:{success,failure,dead_letter}_count`, `:last_delivery` and `:circuit_open`; EndpointMgr's `/status` reads them.

Engine internals, all of which are worth keeping:

| Component | Location |
|---|---|
| Sentinel DCR byte-rate limiter (500 MB/min, warns at 80%) | `forwarder.py:57-79` |
| Per-tenant batch accumulator with drain loop | `forwarder.py:85-178` |
| Batch flush triggers: count, payload bytes, time window | `forwarder.py:141-158` |
| Bounded concurrent flush (semaphore) | `forwarder.py:168-178` |
| Config cache (Redis, 5-min TTL) + DB load | `forwarder.py:209-266` |
| Circuit breaker, success/failure counters | `forwarder.py:288-348` |
| Dead-letter counter (24h rolling) | `forwarder.py:350-360` |
| Connectors | `siem/connectors/{webhook,splunk_hec,sentinel_ingestion}.py` |
| Formatters | `siem/formatters/{json_formatter,cef_formatter}.py` |

### 4.2 Decision: into EndpointMgr, not a separate service

Rationale:

- **The feature is already fire-and-forget and lossy by design.** `forward()` checks the circuit breaker *before* enqueueing and drops events when open (`forwarder.py:383-386`). There is no durable queue; `_record_dead_letter()` increments a counter only. The isolation a separate process would provide is already supplied by the breaker.
- **A separate service would rebuild the currently-broken part.** Config lives in the tenant DB (`_load_config_from_db` → `TenantOrm`, `forwarder.py:243-266`) and the config API is already in EndpointMgr. Splitting the engine from its config surface means building a cross-process invalidation and stats channel — which is exactly what is broken today (§4.3).
- **The event source is already in EndpointMgr.** `alert_processor.py` is the single funnel every agent alert passes through.
- **EndpointMgr already performs third-party egress** — MS Graph, Zitadel, GCS and Account Management, across seven modules using `aiohttp`/`httpx`. This is not a new class of risk in that process.
- **Fewer services is the objective.** A SIEM-only service would be the most idle deployment in the fleet while costing an image, Dockerfile, Helm subchart, cloudbuild stage, healthcheck, nginx entry and secrets wiring.

Destination: `src/endpointmgr/siem/`. Not `shared/`, since EndpointMgr becomes the only consumer.

### 4.3 Pre-existing defects not to carry forward

Four, all visible in the current code:

1. **Cold-cache silent drop.** `forward()` does `if not config: config = self._config_cache.get(tenant_id, (None, 0))[0]` then `if not config or not config.enabled: return` (`forwarder.py:372-375`). `analysis/main.py:638` calls it with **no** config argument, so delivery depends on the cache being warm, and nothing in that path warms it. Events are dropped until something else populates the cache. In the new design the enabled-check is authoritative at drain time, from the DB.
2. **Inert status and breaker in EndpointMgr.** `get_siem_forwarder(redis_client=None)` is a process-local singleton (`forwarder.py:760-768`) and EndpointMgr calls it with no client at all four sites (`siem_settings.py:142,177,192,203`). So in EndpointMgr's process `get_delivery_status()` returns zeros (early return at `:573`), `_is_circuit_open()` is always `False`, and `reset_circuit_breaker()` is a no-op. Co-locating the engine fixes this by giving the singleton a real client.
3. **Config-cache invalidation is a no-op in EndpointMgr.** `invalidate_config_cache()` (`:268-284`) is the cross-process mechanism, and EndpointMgr cannot use it for the same reason. Config saved in the dashboard takes up to the 5-minute TTL to reach the engine.
4. **Unawaited per-event task.** `asyncio.create_task(forward(...))` per event (`analysis/main.py:637`) gives no backpressure and no error surface. Acceptable for fire-and-forget; wrong for durable.

### 4.4 Not designing into a corner (A1)

What determines whether durability is a retrofit or a configuration change is **what holds the record of intent to deliver** — not which process runs the delivery loop. Today that record is an in-process `asyncio.Queue` (`forwarder.py:91`, unbounded), which is why the feature must be lossy.

The asymmetry that helps: **EndpointMgr already persists the alert before forwarding.** `alert_processor.py` writes the row, then enqueues. Analysis had to be lossy because the enriched event existed only in flight; EndpointMgr's events are already durable. Only the *delivery bookkeeping* is in memory.

So the non-cornering design is cheap now:

- **Keep everything below the queue** — batching, connectors, formatters, breaker, DCR limiter. All unaffected.
- **Change what feeds the queue.** Instead of `forward()` being called from the alert-write path, have the drain loop pull: `where siem_delivered_at is null and <tenant siem enabled> order by timestamp limit N ... FOR UPDATE SKIP LOCKED`. That single change buys at-least-once, replay, restart survival and multi-replica safety.

With the seam in Postgres, later extraction to its own deployment is a new entrypoint pointed at the same database — same code, no redesign. The config API stays in EndpointMgr; breaker and stats are already shared via Redis.

**The specific things that would corner you**, all inherited if `forward()` is ported verbatim:

1. **Request-path enqueue into an in-memory queue as the only record.** This is the actual corner; everything else is recoverable.
2. **The cold-cache silent drop** (§4.3.1). A durable design needs an authoritative enabled-check at drain time.
3. **Unawaited `create_task` per event** (§4.3.4).
4. **Breaker-open meaning *drop*.** Durable semantics want *defer* — leave the row unmarked and back off.
5. **Binding the accumulator to request-scoped state.** `forward()` currently takes a plain `tenant_id` string and a dict; keeping that discipline lets the drain loop run outside a request. Do not pass it a tenant-scoped `AsyncSession` or let it read tenant-middleware contextvars.
6. **Lifecycle only in the FastAPI lifespan.** Put start/stop behind a module-level runner that `main.py` calls, so a separate entrypoint can invoke the same thing.

Recommendation under A1: build the outbox drain now. It is a column plus index on data already persisted, a drain query, and a claim. The minimum viable hedge, if deferring, is items 1 and 5 — make the enqueue durable and keep the accumulator context-free. Everything else can be fixed later without moving code between processes.

### 4.5 Precedent in this codebase

The constituent parts are all idiomatic here; the assembled pattern is not.

| Pattern | Established? | Where |
|---|---|---|
| Background worker polls a table, transitions row status | **Yes**, 6+ instances | `tasks/scheduled_cleanup.py:111-211` (closest shape: select pending → execute → mark `executed`/`failed` with error), plus `deprovisioning_worker.py`, `pending_response_expiry.py`, `uninstall_expiry.py`, `agent_reaper.py`, `cleanup.py`, `jobs/usage_aggregation.py` |
| `FOR UPDATE SKIP LOCKED` claiming in a worker | **Yes**, one path | `shared/services/response_queue.py:1082-1114`, `:1102` — driven by `tasks/pending_response_expiry.py` |
| Delivery ledger with re-delivery counting | **Yes**, twice | `shared/db_models.py:310,332-333` (`commands`: `status` + `retry_count`); `:1912,1934-1943` (`uninstall_commands`: `status`, `delivered_at`, `acknowledged_at`, `completed_at`, `error_message`, `delivery_count` — "Story 12.22 re-delivery protocol") |
| Worker that claims rows **and** does outbound third-party I/O with per-row retry | **No** | `scheduled_cleanup` does local work; `deprovisioning_worker` makes external calls one-shot per tenant; `alert_forwarder` retries in memory only |

The outbox composes the first three. Other `with_for_update` sites (`installer_service.py`, `rollout_controller.py`, `tenant_provisioning.py`, `agent_tenant_mapping.py`) are request-path race protection, not worker claiming.

### 4.6 Event shape (A2)

Under A2 the payload is free to be EndpointMgr-native. Two consequences:

- `formatters/json_formatter.py::format_standard_event` and the CEF formatter are written against `StandardEvent` and get retargeted at `alert_processor.py`'s existing serialiser, `_alert_to_dict` (`:550`), which already emits a StandardEvent-shaped payload.
- `forward()` filters on `event_dict.get("type") == "agent_response"` for `config.filter.alerts_only` (`forwarder.py:377-381`); rewrite against EndpointMgr's own alert types.
- Exclude `is_enriched` from the payload (§6.3) — shipping an always-`false` field to a SIEM is misleading.

Enrichment-derived fields (geo, maliciousness, rareness) will be absent. MITRE technique and tactic will also be absent unless the server-side derivation in §6.1 is built — the agent sends `None` for both in production builds. Since these are among the most useful fields a SIEM consumer receives, §6.1 is effectively a prerequisite for the SIEM payload being worth forwarding.

## 5. Dashboard impact

Nav labels are from `src/components/Sidebar.tsx:77-311`.

### 5.1 Removed outright

| Side-nav | Route | Backend it needs |
|---|---|---|
| **COMMAND CENTRE** | `/` | `services/commandCentre.ts:187,282,332,727` → `/api/graph/query` → `getAnalysisPipelineUrl()` (`api/graph/query/route.ts:5,66,123`) → Analysis → Neo4j |
| Anomaly Detection | `/analysis/anomaly` | `services/eagerbeaver.ts:297` → `/api/analysis` → Analysis `/api/v1/pipeline` |
| Selection Logic | `/analysis/selection` | as above |
| Data Sanitisation | `/analysis/sanitization` | `api/sanitisation/rules/route.ts:3,35,96` → `getAnalysisPipelineUrl()` |
| Agent Dynamics | `/responder/agent/dynamics` | `/api/v1/agents/graph` → Responder |
| Agent Config | `/responder/agent/config` | `/api/v1/agent/admin/{config,status,reload}` → Responder |
| Agent Tuning | `/responder/agent/tuning` | `/api/responder/api/v1/agent/langchain/*` |
| Forwarding | `/forwarding/sessions` | `services/forwarding.ts:167` → `/api/v1/forwarding` → EndpointMgr, but the data is relay-written Redis `fwd:sessions:*` — the API survives and returns empty |

**The Command Centre is the landing page and the default route.** Its content is entirely the graph — `src/app/page.tsx` renders `components/CommandCentre/` including `GraphView3D.tsx`, and nothing else. A replacement landing page is part of this work and is the largest single UX consequence.

### 5.2 Removed, not in the side-nav

Reachable by URL:

- `/responder/agent/langchain`, `/responder/activity`, `/responder/automation` — Responder
- `/settings/forwarding/commands`, `/settings/forwarding/environments`, `/settings/forwarding/rules` — relay config
- `/responses` — **already dead code**: `next.config.js:66-79` permanently redirects `/responses` and `/responses/:path*` to `/countermeasures`, so `src/app/responses/page.tsx` is unreachable today

### 5.3 Mixed — must be split, not deleted

**Integrations — `/integrations`.** The SIEM configuration UI lives here, not at `/settings/siem`:

- Dies: the integration tiles (`/api/integrations`, `/api/integrations/{id}/configure`) and `/api/system/license` — both Analysis.
- Survives: `SiemConfigurationModal` and `siemSettingsService` (`src/app/integrations/page.tsx:10,14-17`), including `applySiemStatuses()` (`:38-50`) which reads config plus delivery status per connector tile, via `services/siemService.ts:104,134` → `/api/v1/tenants/{id}/settings/siem`.

`src/app/settings/siem/page.tsx` is a 16-line redirect *into* `/integrations`. Rehoming the SIEM tiles to `/settings/siem` and inverting that redirect reuses a page shell that already exists.

### 5.4 Survives, degraded

**DETECT — `/alerts`.** EndpointMgr-backed and structurally fine, but renders enrichment state: `isActivelyEnriching()` (`:119-121`), the "Enriching" indicator (`:472`), the Enrichment Complete/In progress/N/A field (`:552-553`), and `mitre_tactics[0]` as a column (`:361`). See §6.

**`useGraphUpdates.ts` / `/api/graph/events`.** A false alarm worth recording: that SSE route is a self-contained 112-line local stub with no upstream fetch, so it will not error — it simply has nothing to announce. Remove it with the Command Centre.

**DISRUPT — `/countermeasures`.** EndpointMgr-backed with no dead upstream, and renders the pending-response queue: the list, the approve/reject dialogs and the ramp stages, plus an executed-response history that reads the same table (`src/app/countermeasures/page.tsx:123-124,225-231`). Retiring Responder removes `queue_response()`, the table's only writer (§2.2 has no equivalent HTTP seam because there is none — the coupling is a shared table, not a call), so the page renders empty indefinitely. `/response-queue` is a redirect into it at `?tab=pending` and degrades with it.

Whether an operator-facing initiation path replaces the queue is a product decision, not a retirement task. The dashboard work is the same either way: both halves of the page read approval fields (`reviewed_by`, `response_status`), so repointing them at `commands` is a rewrite rather than a URL change.

> **Correction, 2026-08-06.** §5.5 originally listed `/countermeasures` and `/response-queue` as unaffected. That was right about the API surface and wrong about the page: the signals in §Method cover imports, HTTP calls and deployment wiring, and this coupling is a table one service writes and another reads. Surfaced while scoping the response lifecycle for the rebuild; the same class of defect as the enrichment indicator in §6.2.

### 5.5 Unaffected

- **DISRUPT** Capabilities `/countermeasures/capabilities`, `/countermeasures/anti-agent` — the parent `/countermeasures` degrades, see §5.4
- Decoys `/detections`, Deployment Limits `/detections/quotas` — `services/detections.ts:66` → `/api/v1/endpointmgr`
- **DEPLOY** `/endpoints`, `/endpoints/policies`, Agent Groups, Agent Updates, Agent Defaults
- **PERFORMANCE** `/performance` — `services/performance.ts:96` → `/api/v1/endpointmgr/performance`
- All 10 **Appliance** items (Monitoring, TLS Certificates, Network, Backups, Updates, Data Management, Cluster, Support, SSO, Factory Reset) — liarbirdctl
- All 4 **Settings** items (User Management, License, Privacy, Audit Logs)
- **Admin → System Settings** `/system/settings` and children (`audit-logs`, `response-mode`, `tag-patterns`) — `services/endpointmgr.ts:456`
- login, logout, onboarding, terms, legal, help/docs, admin identity-providers, `/agents/[id]/{capabilities,manifest}`, `/agents/deploy`

### 5.6 Net effect on the nav

- **Main** keeps DETECT (degraded), DISRUPT (degraded), DEPLOY and PERFORMANCE, but loses its landing page, the Integrations child (becomes SIEM-only elsewhere), Data Sanitisation, and DISRUPT's Forwarding child.
- **Appliance** and **Settings**: untouched.
- **Admin** loses 5 of 6 items, leaving only System Settings — worth folding into Settings rather than keeping a one-item group.

### 5.7 Supporting code to delete alongside

- `next.config.js:81-146` rewrites: 3 Responder (`/api/v1/agent`, `/api/v1/agents`, `/api/v1/neo4j`) and 5 Analysis (`/api/analysis`, `/api/v1/pipeline`, `/api/integrations` ×2, `/api/system`)
- `src/app/api/responder/[...path]/route.ts`; `src/app/api/graph/{query,events,clear}/route.ts`; `src/app/api/sanitisation/*`
- `src/services/{responder,eagerbeaver,commandCentre}.ts` and `responder.test.ts`
- `src/components/CommandCentre/`
- `src/middleware.test.ts:141-147` asserts on `/api/graph/*` and `/api/responder/status`

## 6. Alerts UI: MITRE and `is_enriched`

### 6.1 MITRE tactics — derive server-side, do not drop

Analysis was not enriching MITRE data. `custom_parsers/telemetryd_alert.json:112-123` maps `raw_field: "technique"` → `standard_field: "technique"` and `raw_field: "tactic"` → `standard_field: "tactic"` — a **field rename**. The columns exist and are GIN-indexed (`database/init/01-schema.sql:241-243`, `:313-314`), and `alert_processor.py:574` already falls back to the raw payload:

```python
"tactic": (alert.mitre_tactics[0] if alert.mitre_tactics else raw.get("tactic", ""))
```

**The agent does not supply these fields in shipped builds.** Verified against LiarbirdAgent (`3b69263`, 2026-07-27):

- Wire field names are `technique` and `tactic` (`src/alerts.rs:23-24`), with no serde rename — unlike neighbouring fields (`id`→`alert_id`, `source_ip`→`addr_rm`). They match the parser exactly. `mitre_technique` appears only in `src/logging/event_log.rs:167`, on a local Windows event-log struct, not the server payload.
- Both fields are populated through `diag_opt!()`, defined in `src/opsec_macros.rs:26-40` as `Some(...)` under `#[cfg(not(feature = "production"))]` and **`None` under `#[cfg(feature = "production")]`**. The macro's doc comment names this exact case: *"Use for Option<String> fields: technique, tactic, description."*
- Production is the default and shipped build: `Cargo.toml:10` sets `default = ["production"]`; `.github/workflows/build-customer-agent.yml` defaults `build_mode` to `production` (`:71,120,204`) and maps it to the cargo feature at `:206`, with `dev` deliberately excluded from the allowlist (`:203-204`).

This is Story M4-4.11's intent, not an oversight — the stated purpose is to *"prevent binary analysis from revealing detection capabilities, MITRE coverage, or internal architecture."*

Consequences:

- `alert_processor.py:574`'s `raw.get("tactic", "")` yields `""` in production today. Harmless, since nothing consumes it — which is why it went unnoticed.
- Even in dev/test builds the value formats are inconsistent across emit sites: `src/monitor/minifilter_monitor.rs:886` sends compound `"T1018: Remote System Discovery"`; `src/security/integrity_check.rs:1262-1263` sends bare `"T1562"` with kebab-case `"defense-evasion"`; `MitreMapping` (`src/alerts.rs:896-1030`) yields title-case tactics like `"Discovery"`. A straight passthrough into a GIN-indexed `TEXT[]` would need normalising regardless.

**Action:** derive MITRE server-side in EndpointMgr from the detection profile, and keep the UI column. This is also the OPSEC-correct answer rather than a workaround — M4-4.11 exists to stop *binary analysis* revealing MITRE coverage, and a server-side table satisfies the data need while keeping the mapping out of the agent entirely. EndpointMgr already owns detection profiles (`api/detections.py`, `services/detection_manager.py`, migration `002_detection_profiles.sql`).

Two ready sources for the mapping table:

- `services/alert_forwarder.py:52-72` — `_extract_mitre_technique()`, artifact-type → technique (file→T1005, registry→T1012, network→T1046, cached_credential→T1003). Marked deprecated; the logic is still sound.
- LiarbirdAgent `src/alerts.rs:896-1030` — `MitreMapping::for_alert_type()`, a fuller AlertType → (`technique_id`, `technique_name`, `tactic`) table, portable to the server as data.

If the server-side mapping is not built, MITRE has no data source in production and the UI column should be removed along with the enrichment indicators in §6.2. Keeping the column is conditional on doing the derivation.

`mitre_subtechniques` is unmapped either way and stays empty.

### 6.2 "Enriching" indicator — remove

`isActivelyEnriching()` (`src/app/alerts/page.tsx:119-121`) is a time-window heuristic based on `received_at` versus `timestamp`. With no enricher, every alert shows "In progress…" for that window and then settles on "N/A" permanently — a progress indicator that never completes. Remove the helper, the indicator (`:472`) and the Enrichment field (`:552-553`).

### 6.3 `alerts.is_enriched` — keep the column, drop the UI

Keep it: `alerts` is partitioned by month, so dropping a column is a heavier migration than it appears for zero runtime gain — the partitioning has already forced schema compromises elsewhere, including removing the `alert_id` FK from the ORM (§3.8) — a defaulted boolean costs nothing; and with EndpointMgr owning alerts end-to-end and forwarding to SIEMs, lightweight enrichment (geo at minimum) landing here later is plausible, and this is the flag it would use.

Two writes stop being true and should change:

- `alert_processor.py:312` — `existing.is_enriched = False  # Re-trigger enrichment with merged data`. Nothing re-triggers; remove the line and comment.
- Exclude the field from the SIEM event shape (§4.6).

### 6.4 Summary

| Item | Verdict |
|---|---|
| `mitre_tactics` / `mitre_techniques` UI column | **Keep, conditional on §6.1** — the agent sends `None` in production builds, so the values must be derived server-side from the detection profile. Without that derivation, remove the column |
| `mitre_subtechniques` | Indifferent — empty either way |
| "Enriching" indicator + `isActivelyEnriching()` | **Remove** |
| Enrichment Complete/In progress/N/A field | **Remove** |
| `alerts.is_enriched` column | **Keep**; drop the merge-path write, exclude from the SIEM payload |

## 7. Deployment surface

| Surface | State |
|---|---|
| `docker-compose.yml` | EndpointMgr block `:319-428`; `depends_on` gates on redis/neo4j healthy and analysis-service started; six `NEO4J_PASSWORD:?` guards (§3.4) |
| `liarbird-helm/Chart.yaml` | **Already conditional** — `analysis.enabled` (`:38`), `responder.enabled` (`:43`), `forwardingrelay.enabled` (`:58`), `redis.enabled` (`:69`), `neo4j.enabled` (`:74`) |
| `values-gke-saas.yaml` | Already runs `redis.enabled: false` (`:103-104`) and `neo4j.enabled: false` (`:120-121`), so the toggles are exercised |
| `charts/endpointmgr/templates/configmap.yaml` | Hardcodes `ANALYSIS_API_HOST`/`RESPONDER_API_HOST` (`:13-16`) |
| `charts/endpointmgr/templates/deployment.yaml` | Redis/Neo4j env wiring `:220-263`. **No** initContainer waits on redis/neo4j — the ordering risk is Compose-only |
| `nginx/nginx.conf` | Upstreams `:129,134,145-146`; locations `:191,198,223` (§3.3) |
| `docker-compose.prod.yml` | Service list `:87-107` |
| cloudbuild | `cloudbuild.yaml`, `-prod`, `-staging`, `-release`, `-promote`, `-appliance` |
| `appliance/` | §3.8 |
| Tests | 63 of 228 Python test files reference the three services; 26 reference Neo4j; 49 reference Redis (all of which stay under A4). `tests/conftest.py:158` sets `LIARBIRD_NEO4J_PASSWORD` globally |
| `database/` | `database/init/01-schema.sql` and `000_base_schema.sql` hold the Analysis/Responder-owned tables. Migrations are additive and shared; `alerts.is_enriched`, the rareness/statistics tables and `forwarding_*` become orphans |
| `alerts` partitioning | RANGE-partitioned monthly on `timestamp`, hardcoded through 2030-12; the rolling maintenance helper is wired only into Analysis (§3.8) |

## 8. Sequencing

Four independently shippable stages.

**Stage 1 — decouple, with all five services still running.** Nothing user-visible moves; EndpointMgr becomes independently deployable.

1. Move `src/analysis/siem/` → `src/endpointmgr/siem/`; drop the two `COPY src/analysis/...` lines from both Dockerfiles.
2. Make `neo4j_password` optional in `shared/config/secrets.py` (§3.1).
3. Make the `POST /responses` → Analysis forward fail-soft (§3.2).
4. Retire `AlertForwarder` and `alert_processor`'s `forward_queue`.
5. Wire `get_siem_forwarder(redis_client=...)` in EndpointMgr's lifespan, reusing the rate-limiter connection as `redis_metrics` already does — this activates `/status`, the breaker and cache invalidation (§4.3.2, §4.3.3).
6. Drive SIEM forwarding from `alert_processor.py`, at the point that currently enqueues to `forward_queue`, via the outbox seam (§4.4).
7. Add `await forwarder.close()` to lifespan teardown, next to the existing worker stops. Accumulators flush remaining batches on `CancelledError`, so clean shutdown matters.
8. Derive `mitre_tactics`/`mitre_techniques` server-side from the detection profile and populate at ingest (§6.1) — or drop the column and its UI, but decide rather than defer.
9. Move `ensure_alert_partitions_for_next_n_months(session, n=6)` from Analysis's lifespan into EndpointMgr's, preserving the log-at-WARNING-don't-block semantics (§3.8).

**Stage 2 — toggle off before deleting.** Flip `analysis.enabled`, `responder.enabled`, `forwardingrelay.enabled` and `neo4j.enabled` to false in a non-prod values file (leave `redis.enabled` alone) and run the EndpointMgr and agent E2E suites. The toggles already exist and two are already exercised in `values-gke-saas.yaml`. This is the cheapest way to surface coupling that greps miss.

**Stage 3 — delete per-service, in this order.** nginx upstream and location removed in the *same* commit as each service.

1. **forwardingrelay** — most self-contained: 5 EndpointMgr files, the four Redis surfaces (§2.6), the manifest `relay_url` injection, `shared/forwarding_*`.
2. **responder** — zero EndpointMgr imports; only the inbound `POST /api/v1/endpointmgr/response/execute` (which the dashboard also calls, so it survives) and the dashboard pages.
3. **analysis** — largest fan-out.
4. **Neo4j** — the four sites in §2.5, all straight deletions under A3.

**Stage 4 — dashboard.** Per §5: remove the dead surfaces, rehome the SIEM tiles to `/settings/siem` and invert the redirect, replace the Command Centre landing page, fold the one-item Admin group into Settings, and resolve the alerts enrichment UI per §6.

## 9. Open items

| # | Item | Owner decision needed |
|---|---|---|
| 9.1 | ~~Field name telemetryd emits for MITRE~~ — **closed 2026-08-05.** Fields are `technique`/`tactic`, but `diag_opt!()` strips them to `None` in production builds (§6.1) | Decide: build the server-side derivation, or drop the MITRE column and UI |
| 9.2 | Outbox now, or the minimal hedge (§4.4) | Depends on how firm A1 is |
| 9.3 | Replacement landing page for `/` (§5.1) | Product |
| 9.4 | Appliance and cloudbuild scope (§3.8) | Whether this lands in the same programme or follows |
| 9.5 | Agent behaviour against a dead `relay_url` (§3.6) | Verify before disabling the relay in a fleet with forwarding on |
| 9.6 | Orphaned schema — `forwarding_*` tables, rareness/statistics tables (§7) | Drop now or leave; no functional impact either way |

## 10. Documentation to update

All four describe the current four-service architecture:

- `README.md` — the Services Architecture table (`:91-98`), the tech-stack list (`:105`), the Docker service diagram (`:120-145`), the health-check commands (`:34-36`)
- `docs/local-setup.md` — the service/port table (`:67-76`), the health-check block (`:138-142`), the MITRE-seeding note (`:48`), the host-run service instructions (`:166-179`), the test-prerequisites note (`:226`)
- `CLAUDE.md` — the architecture diagram, directory structure, Key Services sections, and the Neo4j graph-schema section
- `docker-compose.yml` — the `# Neo4j (required for agent registration)` comment above the EndpointMgr Neo4j block (§2.5)
