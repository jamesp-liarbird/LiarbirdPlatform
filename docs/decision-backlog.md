# Decision backlog

What must be settled before this repo becomes the working code repository. Clearing this list is the exit criterion for the documentation phase.

**Status values:** `open` — not started · `drafting` — ADR in progress · `proposed` — ADR written, awaiting its promotion trigger · `accepted` — closed · `deferred` — deliberately postponed, with a reason and a revisit trigger · `withdrawn` — not a decision; settled elsewhere, with a pointer.

Numbers are stable. A withdrawn row keeps its number rather than being removed, because investigations, ADRs and capability briefs cite them, and a citation should not have to be chased when a row's status changes.

| # | Decision | Status | Blocked by |
|---|---|---|---|
| 1 | Architecture style — modular monolith vs alternatives | proposed | 8 (for §2.6, §3.7 only) |
| 2 | ADR estate — numbering, prefix, foreign series | open | — |
| 3 | Dropping Neo4j | withdrawn | — |
| 4 | Where agent-submitted alerts go without `analysis` | withdrawn | — |
| 5 | Command/response lifecycle without `responder` | proposed | 1 |
| 6 | SIEM forwarding ownership | open | 1 |
| 7 | Forwarding-relay reference in agent manifests | open | 1, 5 |
| 8 | Control plane / tenant plane separation | proposed | — |
| 9 | Time-keyed tables and retention | proposed | 8 |
| 10 | Identity and tenancy model | open | — |
| 11 | Dashboard scope | withdrawn | — |
| 12 | Language and framework baseline | accepted | 1 (boundary tooling only) |
| 13 | Validation during the documentation phase | open | — |
| 14 | Migration mechanism | open | 9 |

---

## 1. Architecture style

Modular monolith was the working assumption and is now explicitly open. Needs the alternatives assessed on their merits — including what "modular" is enforced by, since a monolith without enforced boundaries is just the PoC with fewer ports.

Gates decisions 4–7 and 12: under a monolith the abandoned service seams become in-process module contracts; under anything else they are network boundaries with their own failure modes. Writing the seams first means writing them twice.

Proposed — [`adr/architecture-style.md`](adr/architecture-style.md): a coarse-grained modular monolith of about five modules with CI-enforced import contracts, deployed as two process roles (`web`, `scheduler`) from one image. Promotes when a Phase 1 vertical slice runs with its §4.2 assertions green.

The baseline finding is that the PoC is a distributed monolith, so the choice was never about process count — it is about what enforces boundaries and what the second module costs. Options C (separable modules), D (service per capability) and E (the baseline) are recorded as reachable upgrades with named triggers rather than as dead ends.

Leans on decision 8 for what enforcement is available, not for its own outcome: §3.7 there establishes that `GRANT`/`REVOKE` is inert under owner topology, and §2.6 that `FORCE ROW LEVEL SECURITY` survives it but is row-scoped. So module-level data ownership is a CI concern under every option, which removes it as a discriminator. Anyone revisiting 8's §2.6 or §3.7 should re-read this.

## 2. ADR estate

Five numbering series, all in AgileFramework: `ADR-001…005` as files (with `005` misfiled outside `docs/adr/`); `ADR-MT-*` and `ADR-VM-*` as subsections inside `AgileFramework/docs/architecture-m3-multi-tenant.md` and `architecture-m3-vm-appliance.md`, the latter carrying a `VM-PATTERN-*` sibling series at the same altitude; and `ADR-M3-*` inside `AgileFramework/docs/epics.md`, cited by stories as `docs/epics-m3.md#ADR-M3-00N`, a path that no longer resolves.

Alongside them sit ~45 unnumbered `Decision N.M` records in the M1, M2 and PoC architecture documents. They carry no IDs and no status fields, and one contradicts a numbered ADR: `archive/m2` Decision 1.1 has the agent MSI identical for all customers with the server address in the registration token, which `ADR-005` reverses on the finding that the URL is compile-time-only and the token's copy is ignored at runtime. Neither is marked superseded.

Restarting at `001` here is the current intent. Open questions: whether this repo's series takes a prefix to avoid colliding with the AgileFramework numbers (the collision is live — the control-plane ADR cites `ADR-001` on Zitadel organisations in its body, and its `Related` line carries a bare `ADR-004` meaning AgileFramework's), and what becomes of the foreign series as their subject matter is re-decided here.

Until this closes, ADRs are named and cross-referenced by slug.

## 3. Dropping Neo4j

Withdrawn — the graph is not a platform component but the internal data model of `analysis` and `responder`, so it leaves with them under the endpointmgr-first constraint, which names it. Nothing in scope reads or writes it, and the PoC's export path already emits its Neo4j-absent archive whenever the graph is empty.

Evidence: [`investigations/2026-08-06-neo4j-scope-exclusion.md`](investigations/2026-08-06-neo4j-scope-exclusion.md). Removing the client from the PoC is that repo's housekeeping. What the graph's correlation was doing, and whether anything replaces it, is settled in [`rebuild/capabilities/alert-ingestion.md`](rebuild/capabilities/alert-ingestion.md) (§4).

## 4. Where agent-submitted alerts go without `analysis`

Withdrawn — they stay where they already are. Alert ingestion, storage, the read surface and `event_key` deduplication are all endpointmgr's today, and the dropped service enriched a row the ingest path had already written. There is no seam to place and no owner to find, so what remains is implementation guidance rather than a decision.

Guidance: [`rebuild/capabilities/alert-ingestion.md`](rebuild/capabilities/alert-ingestion.md) — what the alert record stops carrying and why, the ingest path's acknowledge-without-persist defect, and the handoffs to decisions 5, 6 and 9. Correlation is settled there too: the graph-modelled kind leaves with the graph (§3), and deduplication never left.

Enrichment's eventual return is not this row's business either. It arrives as a module under decision 1, with a schema change of its own, rather than by writing columns reserved in advance.

## 5. Command/response lifecycle without `responder`

Responses execute on the endpoint by one of two paths, selected by the manifest's per-asset response mode. Server-commanded ramping follows the schedule in the command, or a fixed default ladder when none is supplied. Autonomous response is the randomised one, drawing delay, duration and intensity per execution from bounds compiled into the agent (`LiarbirdAgent/src/response/immediate.rs`) so the response is not fingerprintable as a security control. `alert_only` suppresses both; `hitl` suppresses both and waits for a server command.

Agents call `agent/responses` and `response/command/{id}/acknowledge`, commands ride the heartbeat, and execution continues while disconnected — progress reports queue and flush on reconnect. Server-side response state is therefore a lagging report rather than a control surface. `response.py`, `response_queue.py` and `shared/services/response_queue.py` split along that line: the queue, HITL gate, expiry sweep and agent-facing paths are endpointmgr's, response *selection* is the responder's.

The producer goes with selection. `queue_response()` is called only from `responder/services/hitl_workflow.py` and `responder/services/langgraph_agent.py`, and no route creates a pending response, so an operator cannot initiate one. That queue is also the whole operator surface — the disrupt page is its list, approvals and history — so what is at stake is the capability, not a table.

Proposed — [`adr/operator-initiated-endpoint-responses.md`](adr/operator-initiated-endpoint-responses.md): an operator dispatches against the existing command route, the approval queue does not come across, and `hitl` means an operator decides. Promotes when the disrupt capability ships with its §4.3 assertions green. Implementation guidance: [`rebuild/capabilities/response-lifecycle.md`](rebuild/capabilities/response-lifecycle.md).

Deciding nothing is not neutral, which is what settles it: with the mode assigned per asset, `immediate` assets keep responding autonomously while `hitl` assets respond never and record nothing. Response-mode configuration (`detection_profile_response_mode`, `response_mode_critical_immediate`) selects between the paths and stays endpointmgr's under any outcome.

## 6. SIEM forwarding ownership

`endpointmgr/api/siem_settings.py` imports `analysis.siem.config` and `analysis.siem.forwarder` directly — the one hard Python import crossing the boundary being dropped. Either SIEM forwarding comes across as its own module, or the settings CRUD comes across without a forwarder behind it.

## 7. Forwarding-relay reference in agent manifests

`manifests.py` hands agents a `FORWARDING_RELAY_URL`. With `forwardingrelay` out of scope, either the manifest field goes (an agent-visible change, so check it against the wire contract) or something answers on that URL.

The manifest also carries `response_config` (wire name `r_cfg`, holding `permitted_actions`), which is the agent's response envelope. If decision 5 lands on endpoint-autonomous responses, that field is their only control surface, so manifest work has to keep it — and it currently fails open, since `LiarbirdAgent/src/response/immediate.rs` treats an absent action key as permitted.

The envelope has a second half the server never sends. `LiarbirdAgent/src/agent_config.rs` defines `ir_cfg` for the bounds the autonomous path randomises within — delay, duration, intensity and ramp-up — and `manifests.py` emits no such field, so those responses run on compiled-in defaults. Whether the server should tune them follows decision 5.

## 8. Control plane / tenant plane separation

ADR carried over from the PoC repo — [`adr/control-plane-tenant-plane-separation.md`](adr/control-plane-tenant-plane-separation.md). Proposed; promotes once a Phase 1 vertical slice confirms its §8.1 gates. Its §2.2 and §2.4 already answer the dual-topology standing constraint in detail, so it is the baseline other storage decisions build on rather than an open question.

One known revision: its §1.1 attributes the duplicated bootstrap SQL to `EagerBeaver/` being a separate Docker build context. Decision 1 may delete that cause, and the argument needs re-reading when it does.

## 9. Time-keyed tables and retention

ADR carried over — [`adr/time-keyed-tables-with-bounded-retention.md`](adr/time-keyed-tables-with-bounded-retention.md). Proposed; promotes when the schema lands with its invariants under test. Depends on decision 8 for storage topology.

Carries a live risk out of the PoC: the partition roller runs in the Analysis service lifespan, and Analysis is being dropped. Removing it without resolving this re-arms the P0 that ISSUE-212 raised.

## 10. Identity and tenancy model

AgileFramework's `ADR-001` establishes tenant = Zitadel organisation, and the control-plane ADR depends on it — without an org and a member there is no JWT carrying a tenant claim. Whether that model carries across unchanged, and whether Zitadel remains the identity provider, has not been revisited for the rebuild. It underpins decision 8, so an unexamined assumption here propagates.

## 11. Dashboard scope

Withdrawn — the dashboard is in scope and is carried across rather than rebuilt, with the surfaces the descope breaks rebuilt and the rest copied as-is and validated by a testing pass. There is no fork left to argue: the route-by-route sort already exists as evidence, and the one contested case is settled by decision 5.

Guidance: [`rebuild/dashboard-carry-over.md`](rebuild/dashboard-carry-over.md) — what is rebuilt, what is removed, and why no structural conditions are imposed on the carried code.

The API surface question this row used to carry is answered rather than open: paths do not change. The agent wire contract pins the `/api/v1/endpointmgr/…` prefix permanently, so re-cutting the dashboard's along module lines would leave one server answering two prefixes.

## 12. Language and framework baseline

Accepted — [`adr/language-and-framework-baseline.md`](adr/language-and-framework-baseline.md): Python 3.14 / FastAPI / SQLAlchemy 2.x async / Pydantic 2.x / pytest / import-linter, with majors decided in the ADR and everything below them left to the lock file. Accepted rather than proposed because the row has no empirical content a vertical slice could falsify; the interpreter number is the one checkable part, and §4.1 makes a failure select 3.13 under the same criterion rather than reopen the decision.

The interpreter moves rather than carrying across, because it currently has no single value to carry: the lock file was compiled on 3.12, the images run 3.11, and nothing declares a requirement. Fixing that means choosing, and the criterion chooses the newest release still in bugfix support.

The dashboard half of this row needs no decision. It is carried as-is under no structural conditions (decision 11, [`rebuild/dashboard-carry-over.md`](rebuild/dashboard-carry-over.md) §1), so its Next.js and TypeScript baseline follows from the carry rather than from a fork.

Still blocked by decision 1 for the boundary-tooling clause only: the ADR names import-linter, and the contracts behind decision 1's §4.2 assertions 1 and 2 track that decision. Standing it up is build work — [`rebuild/toolchain-baseline.md`](rebuild/toolchain-baseline.md) §5 — and rule 2 of the carry-over rules is the rule currently resting on a check that does not exist.

## 13. Validation during the documentation phase

Both carried-over ADRs are deliberately stuck at Proposed pending implementation evidence. If the documentation phase produces no code, nothing can be promoted and "documentation complete" has no achievable definition.

Either the phase ends with a stack of Proposed ADRs that a Phase 1 vertical slice promotes in a batch, or throwaway spikes are permitted during the phase specifically to close validation gates. Decisions 8 and 9 have empirical gates — role-level `search_path` behaviour under a pooler, whether the provider seam absorbs both deployment shapes — that a few hundred lines would settle far more cheaply than discovering the answer mid-build.

## 14. Migration mechanism

Schema changes are applied by hand-rolled SQL: numbered files in `database/migrations/` with paired `_down.sql` rollbacks, a home-made tracking table (`001_migration_tracking.sql`), a `run_migrations.py` runner and its own image (`database/Dockerfile.migrations`). Alembic is a *declared* dependency that nothing uses — `alembic>=1.12.0` at `EagerBeaver/src/requirements.in:47`, with no `alembic.ini` and no `env.py` anywhere — and it is carrying a CVE pin on transitive `mako` (ISSUE-354), so the unused framework has an active maintenance cost. Removing the dependency is the proof-of-concept repo's housekeeping.

The fork is which mechanism owns the schema, and it is not a "preserve what exists" question in either direction: the incumbent is a runner someone wrote, and the alternative is a framework already paid for and never wired up.

Blocked by decision 9, which determines what the mechanism has to be able to express. Time-keyed tables need partitions created ahead of use, and that decision also carries the unresolved partition roller — a migration mechanism that cannot create partitions leaves the roller as the only thing that can, which is how ISSUE-212 arose.
