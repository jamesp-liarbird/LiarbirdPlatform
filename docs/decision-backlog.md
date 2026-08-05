# Decision backlog

What must be settled before this repo becomes the working code repository. Clearing this list is
the exit criterion for the documentation phase.

**Status values:** `open` — not started · `drafting` — ADR in progress · `proposed` — ADR written,
awaiting its promotion trigger · `accepted` — closed · `deferred` — deliberately postponed, with a
reason and a revisit trigger.

| # | Decision | Status | Blocked by |
|---|---|---|---|
| 1 | Architecture style — modular monolith vs alternatives | open | — |
| 2 | ADR estate — numbering, prefix, foreign series | open | — |
| 3 | Dropping Neo4j | open | — |
| 4 | Where agent-submitted alerts go without `analysis` | open | 1 |
| 5 | Command/response lifecycle without `responder` | open | 1 |
| 6 | SIEM forwarding ownership | open | 1 |
| 7 | Forwarding-relay reference in agent manifests | open | 1, 5 |
| 8 | Control plane / tenant plane separation | proposed | — |
| 9 | Time-keyed tables and retention | proposed | 8 |
| 10 | Identity and tenancy model | open | — |
| 11 | Dashboard scope | open | — |
| 12 | Language and framework baseline | open | 1 |
| 13 | Validation during the documentation phase | open | — |

---

## 1. Architecture style

Modular monolith was the working assumption and is now explicitly open. Needs the alternatives
assessed on their merits — including what "modular" is enforced by, since a monolith without
enforced boundaries is just the PoC with fewer ports.

Gates decisions 4–7 and 12: under a monolith the abandoned service seams become in-process module
contracts; under anything else they are network boundaries with their own failure modes. Writing
the seams first means writing them twice.

## 2. ADR estate

Four numbering series across two repos: `ADR-001…005` in AgileFramework (with `005` misfiled
outside `docs/adr/`), and `ADR-MT-*`, `ADR-M3-*`, `ADR-VM-*` existing only as `####` subsections
inside `AgileFramework/docs/architecture-m3-multi-tenant.md` and `architecture-m3-vm-appliance.md`.

Restarting at `001` here is the current intent. Open questions: whether this repo's series takes a
prefix to avoid colliding with the AgileFramework numbers (the collision is live — the
control-plane ADR's body cites AgileFramework's `ADR-001` on Zitadel organisations), and what
becomes of the foreign series as their subject matter is re-decided here.

Until this closes, ADRs are named and cross-referenced by slug.

## 3. Dropping Neo4j

Stated as a scope decision at the outset but never argued, and it is load-bearing beyond deleting a
client. Neo4j appears in only five endpointmgr files, but two are `tenant_provisioning.py` — where
it is a step in the provisioning state machine (`STEP_NEO4J_DB`) — and `tenant_deprovisioning.py`,
where it is an export step in the tenant data-extraction path.

So this changes the tenant lifecycle and the data-export obligation, not just the dependency list.
Needs a short ADR recording what the graph was for, what is lost, and what if anything replaces it.

## 4. Where agent-submitted alerts go without `analysis`

Agents `POST` to `/api/v1/endpointmgr/agent/alerts`, and endpointmgr forwards to the Analysis
service over `ANALYSIS_API_HOST`/`PORT` from `alert_forwarder.py`, `alert_processor.py` and
`agent_registration.py`. With Analysis out of scope, alert ingestion, normalisation and storage
need an owner. The agent-facing path cannot change (standing constraint), so this is a question
about what sits behind it.

## 5. Command/response lifecycle without `responder`

Agents call `agent/responses` and `response/command/{id}/acknowledge`. The `response.py` and
`response_queue.py` routers, and the queue in `shared/services/response_queue.py`, span the
endpointmgr/responder boundary. Which half survives, and what drives responses in the absence of
the AI responder, is undecided.

## 6. SIEM forwarding ownership

`endpointmgr/api/siem_settings.py` imports `analysis.siem.config` and `analysis.siem.forwarder`
directly — the one hard Python import crossing the boundary being dropped. Either SIEM forwarding
comes across as its own module, or the settings CRUD comes across without a forwarder behind it.

## 7. Forwarding-relay reference in agent manifests

`manifests.py` hands agents a `FORWARDING_RELAY_URL`. With `forwardingrelay` out of scope, either
the manifest field goes (an agent-visible change, so check it against the wire contract) or
something answers on that URL.

## 8. Control plane / tenant plane separation

ADR carried over from the PoC repo — [`adr/control-plane-tenant-plane-separation.md`](adr/control-plane-tenant-plane-separation.md).
Proposed; promotes once a Phase 1 vertical slice confirms its §8.1 gates. Its §2.2 and §2.4 already
answer the dual-topology standing constraint in detail, so it is the baseline other storage
decisions build on rather than an open question.

One known revision: its §1.1 attributes the duplicated bootstrap SQL to `EagerBeaver/` being a
separate Docker build context. Decision 1 may delete that cause, and the argument needs re-reading
when it does.

## 9. Time-keyed tables and retention

ADR carried over — [`adr/time-keyed-tables-with-bounded-retention.md`](adr/time-keyed-tables-with-bounded-retention.md).
Proposed; promotes when the schema lands with its invariants under test. Depends on decision 8 for
storage topology.

Carries a live risk out of the PoC: the partition roller runs in the Analysis service lifespan, and
Analysis is being dropped. Removing it without resolving this re-arms the P0 that ISSUE-212 raised.

## 10. Identity and tenancy model

AgileFramework's `ADR-001` establishes tenant = Zitadel organisation, and the control-plane ADR
depends on it — without an org and a member there is no JWT carrying a tenant claim. Whether that
model carries across unchanged, and whether Zitadel remains the identity provider, has not been
revisited for the rebuild. It underpins decision 8, so an unexamined assumption here propagates.

## 11. Dashboard scope

Not yet discussed. The PoC dashboard is ~104k lines of Next.js/TypeScript — larger than endpointmgr
and `shared` combined. Whether it is in scope for the rebuild at all, rebuilt, or carried across,
changes what the API surface must serve beyond the agent contract.

## 12. Language and framework baseline

Python 3.11 / FastAPI / SQLAlchemy is the PoC's stack and the implicit assumption for the rebuild,
but has not been recorded as a decision. Worth an explicit — and probably short — ADR rather than
inheriting by default, since the modular-monolith question (decision 1) touches how boundaries are
enforced, which is partly a language-tooling matter.

## 13. Validation during the documentation phase

Both carried-over ADRs are deliberately stuck at Proposed pending implementation evidence. If the
documentation phase produces no code, nothing can be promoted and "documentation complete" has no
achievable definition.

Either the phase ends with a stack of Proposed ADRs that a Phase 1 vertical slice promotes in a
batch, or throwaway spikes are permitted during the phase specifically to close validation gates.
Decisions 8 and 9 have empirical gates — role-level `search_path` behaviour under a pooler, whether
the provider seam absorbs both deployment shapes — that a few hundred lines would settle far more
cheaply than discovering the answer mid-build.
