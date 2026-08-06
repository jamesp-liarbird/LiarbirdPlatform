# Alert ingestion

Agents submit detections; the server stores them; operators read them. Descoping `analysis` removes
enrichment from that path and nothing else — ingestion, storage, the read surface and deduplication
are already `endpointmgr`'s, so most of this capability carries across working.

What follows is the delta: what holds, what the wire contract fixes, what changes, and what the alert
record stops carrying. The cross-cutting structural changes are in
[`../carry-over-rules.md`](../carry-over-rules.md); §6 notes which of them land hardest here.

## 1. What already holds

Carry these across on their behaviour, subject to §6:

- **The ingest handler** (`endpointmgr/api/agent_registration.py:2425-2466`) — authenticates, binds
  the JWT subject to the batch's agent ID, rejects revoked agents, delegates, answers.
- **The batch processor** (`endpointmgr/services/alert_processor.py:217-363`) — field extraction into
  indexed columns, severity validation against a fixed set, process-hash promotion out of nested
  detail, deduplication.
- **The read surface** (`endpointmgr/api/alerts.py`) — list, filtered query, export, SSE stream,
  statistics, fetch-by-id, acknowledge, dismiss, and the batch forms, across fifteen route
  decorators.
- **The response serialiser** (`services/alert_processor.py:550-623`) and its detail synthesiser
  (`:625-696`), which reconstructs nested process and network context from the agent's flat fields.
  Minus the enrichment and MITRE keys (§4).

## 2. Fixed by the wire contract

Agents in the field constrain these; none may change.

- `POST /api/v1/endpointmgr/agent/alerts`, batched.
- The JWT subject must match the batch's declared agent ID — a mismatch is 403, not a warning.
- Revoked agents are refused before their alerts are read.
- The response body is `{"status": "accepted", "count": N}`. **No per-alert identifiers**, which is
  what makes §3.1 possible without touching the agent.

## 3. What changes

### 3.1 Ingest commits before it acknowledges

A persist failure is currently caught, logged, and not propagated: the loop continues to the next
alert and the handler still answers `accepted` (`services/alert_processor.py:340-341` with
`api/agent_registration.py:2466`). An agent receiving that acknowledgement has no reason to retry, so
a transient database error is silent alert loss on the agent-facing critical path.

Commit the batch, then answer. On failure, return a status the agent retries on. Because the response
carries a count rather than identifiers (§2), this is a status-code change inside a shape the field
already accepts.

### 3.2 Nothing rewrites the record after it is written

Alerts are currently written in two phases — inserted with `is_enriched=False` for another process to
finish and flip (`services/alert_processor.py:335`), with a deduplication merge resetting the flag to
re-trigger enrichment that may never arrive (`:312`). It is a workflow carried in a column, with no
orchestrator, no timeout and no compensation.

Drop the second phase entirely: the in-process forward queue (`:77,352`), the worker draining it
(`:91-129`) and the forwarder it drives (`services/alert_forwarder.py`). The alert is complete when
it is written.

**The trap this closes.** The enrichment update overwrote `raw_event` wholesale with a normalised
event dump (`analysis/alert_storage.py:202-219`), while the read path reaches into that same column
for the agent's own field names to rebuild process context (`services/alert_processor.py:625-696`).
The column therefore holds two different shapes depending on whether enrichment had run. Store the
agent payload as reported, under the agent's field names, and never rewrite it.

### 3.3 Deduplication stays in the ingest path

Alerts sharing an `event_key` merge into one row — non-null fields folded in, detail dictionaries
merged, and constituent event IDs accumulated under `merged_event_ids` for forensic completeness
(`services/alert_processor.py:276-315`). This is the only correlation that survives the descope, and
it is what stops one attacker interaction becoming a row per emitted event.

Keep it inline and synchronous. Moving it downstream reintroduces §3.2.

Drop the `is_enriched` reset at `:312` with the column (§4).

### 3.4 The serialiser loses its dead fields

`_alert_to_dict` currently emits enrichment status, MITRE arrays and three AI-analysis fields, the
last of which are serialised to empty strings on every response
(`services/alert_processor.py:599-613`). All of them go with §4.

Dashboard surfaces reading them go too: the enrichment progress indicator and its
`isActivelyEnriching()` time-window heuristic (`dashboard/src/app/alerts/page.tsx:119-121,472,552-553`),
and the MITRE tactic column (`:361`). The indicator is worth naming as a defect and not just a
removal — with no enricher it shows "In progress…" for its window and then "N/A" permanently, a
progress display that can never complete.

## 4. What the record no longer holds

All of it follows the `analysis` descope. Every column below was audited against the API, repository
and dashboard trees before removal:

| Dropped | Why it goes |
|---|---|
| `entities_ip`, `entities_domain`, `entities_user`, `entities_process`, `entities_file` and two GIN indexes over them (`01-schema.sql:255-259,315-316`) | Entity extraction was the enricher's. No reader anywhere in the estate — the columns were write-only |
| `detection_method` (`:246`) | Written by the enricher, read by nothing |
| `is_enriched`, `enrichment_data`, `enrichment_completed_at` (`:247-252`) | The two-phase write's state (§3.2). Its only reader was the indicator in §3.4 |
| `mitre_tactics`, `mitre_techniques`, `mitre_subtechniques` and their GIN indexes (`:241-243,313-314`) | No production data source — see below |
| `ai_analysis`, `ai_summary`, `ai_recommended_actions`, `ai_analysis_requested`, `ai_analysis_completed`, and the partial index over the last two (`:262-266,320`) | Never written by anything. `AlertRepository.mark_as_analyzed` (`shared/repositories/alert_repository.py:178-199`) has no callers, nothing sets `ai_analysis_requested`, and the index therefore covers no rows. Whether automated analysis returns is decision 5's |
| `neo4j_synced`, `neo4j_synced_at` (`:288-289`) | Follows the graph out of scope |

**MITRE deserves its reason stated in full,** because the fields exist on the wire and look
carryable. The agent's alert struct declares `technique` and `tactic`
(`LiarbirdAgent/src/alerts.rs:23-24`) and the server maps them straight through — but both are
populated via `diag_opt!()`, which expands to `Some(...)` only under
`#[cfg(not(feature = "production"))]` and to `None` otherwise
(`LiarbirdAgent/src/opsec_macros.rs:26-41`). Production is the default feature and the shipped build
(`LiarbirdAgent/Cargo.toml:10`), so **every alert from a deployed agent carries `None` for both.**
This is deliberate — the macro's documentation names these fields, and the intent is to stop binary
analysis revealing detection and MITRE coverage.

So a MITRE value must be derived server-side or not exist. Deriving it from the detection profile is
feasible — two mappings exist to seed a table, at `endpointmgr/services/alert_forwarder.py:52-72` and
`LiarbirdAgent/src/alerts.rs:899` — and was **decided against**: the derived value would be a function
of the detection profile the alert already references, storing a join result in two indexed array
columns, and nothing queries alerts by technique. `AlertRepository.get_by_mitre_technique`
(`shared/repositories/alert_repository.py:127-141`) has no callers either.

Should the alerts surface need a classification axis beyond severity and alert type, hang it on the
detection profile rather than reinstating these columns.

**Do not reserve any of the above "for when enrichment returns."** Enrichment is deferred, its
returning shape is unknown, and a defaulted column on this table is not free. It arrives as a module
with its own schema change.

## 5. Handoffs

- **SIEM forwarding (decision 6).** The alert write is the single funnel every agent alert passes
  through, and §3.1 makes the row durable before the agent is told so. That is what makes a delivery
  outbox cheap here: the event is already persisted, so only delivery bookkeeping is new. The
  forwarder reads alerts through `endpoints`' public surface, not the table
  ([`../carry-over-rules.md`](../carry-over-rules.md) rules 3 and 4).
- **Retention and table shape (decision 9).** That decision owns whether `alerts` is partitioned and
  what purges it — nothing purges it today. Two things not to carry across ahead of it: the monthly
  range partitioning, and the partition-maintenance helper that ran in the descoped service's
  lifespan. Removing the second while keeping the first re-arms a resolved P0 (ISSUE-212) with a
  fuse set at the end of the hardcoded partition coverage.
- **Automated analysis (decision 5).** The `ai_*` columns leave with §4. That decision is free to
  bring automated analysis back with storage of its own; it is not constrained to these columns.

## 6. Carry-over rules that land hardest here

- **Rule 1 (connections).** Already satisfied — `AlertProcessor` is stateless with respect to
  connections and every method takes a session from its caller
  (`services/alert_processor.py:65-72`). It is the positive example the rule cites; keep it that way.
- **Rule 2 (repositories).** Not satisfied. `query_alerts` builds its own filtered query
  (`:396-462`) despite the class holding an `AlertRepository`, as do `get_alerts_for_agent`
  (`:476-498`) and `get_unprocessed_alerts` (`:500-516`). These move into the repository. Note the
  processor's `get_unprocessed_alerts` filters `status == "new"` while the repository method of the
  same name filters on the AI-analysis flags — only the first has a caller
  (`endpointmgr/api/dashboard.py:375`), and only the first survives §4.
- **Rule 4 (table ownership).** The `Alert` model moves out of the shared layer and is declared by
  `endpoints`, in the tenant plane.
- **Rule 6 (grouping).** The ingest handler currently sits inside a 2,656-line registration module.
  Alert ingest is not agent registration; separate them on the way in.

## 7. Related

- [`../carry-over-rules.md`](../carry-over-rules.md) — the structural changes that apply to all
  carried code
- [`../../adr/architecture-style.md`](../../adr/architecture-style.md) — `endpoints` owns alerts, and
  §4.3's requirement that a capability name its owning module and public surface
- [`../../adr/time-keyed-tables-with-bounded-retention.md`](../../adr/time-keyed-tables-with-bounded-retention.md)
  — the alerts table's retention and partitioning, and §7's finding that agent-reported detections
  become the only source of rows
- [`../../investigations/2026-08-05-endpointmgr-only-retirement-audit.md`](../../investigations/2026-08-05-endpointmgr-only-retirement-audit.md)
  — the coupling map this brief's §3 and §4 rest on. Its §6.1 and §6.3 recommend deriving MITRE and
  keeping `is_enriched`; both were advice for retiring inside the running system, where dropping a
  column from a partitioned table is expensive, and §4 decides both the other way
- [`../../investigations/2026-08-06-neo4j-scope-exclusion.md`](../../investigations/2026-08-06-neo4j-scope-exclusion.md)
  — why graph-modelled correlation leaves rather than being rebuilt
