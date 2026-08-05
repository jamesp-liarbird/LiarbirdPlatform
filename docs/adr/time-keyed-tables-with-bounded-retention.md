# Time-keyed tables with bounded retention are unpartitioned by default

**Status:** Proposed — moves to Accepted when the rebuild's schema lands with the §6 invariants under test
**Date:** 2026-08-05
**Deciders:** Platform / Engineering
**Scope:** LiarbirdServer rebuild (`database/`, EagerBeaver `shared/db_models.py` and `shared/repositories/`), `appliance/liarbirdctl` (`src/retention/`)
**Related:** [Control plane / tenant plane separation](control-plane-tenant-plane-separation.md) — §2.2 storage topology, §2.13 fleet migrations
**Supersedes:** the implicit partitioning decision in `database/init/01-schema.sql:300`, carried unchanged from the initial schema commit and never recorded

---

## 1. Context

### 1.1 What exists today

The proof-of-concept schema range-partitions exactly one table by month:

```sql
) PARTITION BY RANGE (timestamp);   -- database/init/01-schema.sql:300
```

Declared four times across the mirrored trees (`database/init/01-schema.sql:300`,
`database/migrations/000_base_schema.sql:206`, and both copies under
`EagerBeaver/src/shared/tenant/bootstrap/sql/`). It is the only `PARTITION BY` in 92 migrations plus
the init schema. Nothing else in the schema is partitioned, and no SQLAlchemy model declares
`postgresql_partition_by`.

**No rationale was ever recorded.** The justification is one comment — *"Partitioned by month for
performance as data grows"* (`01-schema.sql:209-210`) — with no volume estimate, no retention
requirement, and no ADR. It arrived with the initial `data/alerts/*.json` → Postgres migration and
was never revisited.

**Neither benefit of partitioning was realised.**

- *Bulk expiry:* retention is a batched `DELETE` + `VACUUM` (`appliance/liarbirdctl/src/retention/postgres.rs:58`),
  never `DROP PARTITION`. With `alerts_days: 90` (`src/config/types.rs:229`) a live deployment holds
  ~4 months of data in 84 partitions, so whole-partition expiry was available and unused. The
  `DELETE` itself qualifies on `id` alone with `timestamp` confined to a subquery
  (`postgres.rs:162-166`), so it does not prune either — the fix that closed ISSUE-296 restored
  pruning for the row-finding half only.
- *Partition pruning:* 7 of 12 query methods in `shared/repositories/alert_repository.py` carry no
  `timestamp` predicate, as does the unbounded path in `alert_processor.query_alerts`
  (`:413-431`). Each plans across all 84 partitions, takes 84+ relation locks, and merges 84 index
  scans to satisfy `ORDER BY timestamp DESC LIMIT n`.

**The costs were paid in full.** ~19 indexes × 84 partitions ≈ 1,600 index relations per tenant
database; no table may FK to `alerts` on a single column, so the `alert_id` FK was stripped from the
ORM (`story-m3-10.1.md:90,230`); `Base.metadata.create_all()` racing migrations produced *"alerts is
not partitioned"* and migrations were disabled in staging as a workaround
(`story-m3-1.3.md:482-509`); and a hardcoded partition window ran out, producing P0 ISSUE-212 and a
partition roller wired into the Analysis service lifespan — a service now slated for retirement,
whose removal re-arms the same P0.

**The same programme declined to partition the better candidate.** `story-m2-6.6.md:514`, on
`audit_logs`: *"Partition by month for large deployments (documented, not implemented in this
story)."* `audit_logs` is genuinely append-only (migration `069` blocks `UPDATE`/`DELETE` for
non-superusers), has `event_time TIMESTAMPTZ NOT NULL` (`019_audit_logs.sql:20`), and retains for
365 days — four times `alerts`. So one table in the class was partitioned and a stronger candidate
was not, with no principle distinguishing them. Two documents also describe partitioning that does
not exist: `database/README.md:78` claims `agents` is partitioned, and `01-schema.sql:630` carries a
`-- Partition audit_log by month` comment above an unpartitioned table whose name is not even the
live one.

### 1.2 Why this matters now

The rebuild reaches this decision with no data to migrate, so the choice is free — and it is about to
be made four times over, for the four tables in the same lifecycle class, with the same absence of
policy that produced the split above.

Two properties raise the stakes beyond the PoC's:

**Storage topology multiplies the choice.** The control-plane ADR §2.2 makes SaaS database-per-tenant
(`tenant_<slug>` × N). Partitioning is therefore chosen once and instantiated per tenant: the PoC's
layout is ~1,600 index relations *per tenant database*, and a schema change to a partitioned tenant
table is a fleet operation (control-plane ADR §2.13), not one migration.

**The volume premise is unmeasured, and the mechanism that set it is being removed.** In the PoC,
`alerts` receives decoy triggers plus a rareness tail: `main.py:507` admits an event only if
`should_trigger_enrichment` passes (`main.py:793-852` — `endpoint_deception` type, a non-empty
`alert_type`, a `lb_active_*` log type, or any rareness field `> 0.95`), then dedupes by
`detection_hash`. Normalized telemetry is never persisted — `StandardEvent` is a Pydantic model used
only inside the Analysis pipeline and maps to no table. The PoC's stated NFR of *"1000+
alerts/second"* (AgileFramework `architecture.md:390`) is 86M rows/day and is contradicted by that
filter.

The rebuild narrows the source further: retiring Analysis retires rareness scoring and every non-agent
ingest path outright, leaving agent-reported detections as the only writer (§7). Neither the PoC's
rate nor the rebuild's has been measured, and §7 states what to measure.

---

## 2. Decision

### 2.1 The class is defined by lifecycle, not volume

A table is **time-keyed with bounded retention** if all three hold:

1. it has a `NOT NULL` timestamp column that is set at insert and not subsequently changed;
2. rows are insert-mostly — never mutated, or mutated only through a bounded lifecycle that does not
   touch the time column;
3. an operator-configurable retention period deletes old rows.

Volume is deliberately *not* part of the definition. Volume is unknowable when the schema is written
— assuming it is what produced §1.1 — whereas all three properties above are readable from the DDL
on day one and assertable in CI. Volume enters this ADR only as the §2.4 threshold.

**Register.** Every member, its time column, and its retention default:

| Table | Plane | Time column | Retention default | Partitioned |
|---|---|---|---|---|
| `alerts` | tenant | `timestamp` | 90 days (max 3650) | No |
| `audit_logs` | one per lineage (control-plane ADR §2.7) | `event_time` | 365 days | No |
| `usage_metrics` | control | `metric_date` (`DATE`) | 13 months | No — bounded by construction |
| `forwarding_session_history` | tenant | `ended_at` | configurable | No |

Two entries carry qualifications. `usage_metrics` is an hourly aggregate with a
`UNIQUE (tenant_id, metric_date, metric_hour)` upsert key, so its size is bounded by
tenants × hours × 13 months regardless of activity — it can never reach a §2.4 threshold, and its
current hour's rows are mutable, so it satisfies §2.1 only loosely. It is listed for completeness, not
as a candidate. `forwarding_session_history` carries two `NOT NULL` timestamps; `ended_at` is the
retention column, `started_at` serves range queries, and a conversion would have to pick one as the
partition key.

Adding a table to the schema that meets §2.1 means adding a row here. That is the artefact whose
absence made the PoC's split invisible.

### 2.2 Default: not partitioned

Every member is a plain table with a `btree` index on its time column. Retention is a scheduled
`DELETE` filtered on that column.

The stated assumption is that per-tenant volumes for all four members sit two or more orders of
magnitude below where partitioning pays for itself — below roughly 50M rows or 100 GB, where a
`btree` on the time column serves range queries without the planning, locking, and catalog cost of a
partition set. This assumption is **explicitly not yet measured**; §7 states what would falsify it
and §2.4 states what happens then.

For `alerts` specifically, one further property argues against partitioning independently of volume:
rows are updated repeatedly through triage (`status`, `dismissed_*`, `ai_analysis*`,
`is_false_positive`, `resolved_at`). Partitioning assumes append-only time-series; a row whose time
column ever changes must be physically moved between partitions.

### 2.3 Invariants every member carries, partitioned or not

These are what keep §2.4 a cheap option rather than a rewrite. They cost nothing in an unpartitioned
table.

1. **The time column is `NOT NULL`** and immutable after insert. A nullable column cannot be a
   partition key.
2. **The primary key and every natural-key unique constraint include the time column** — e.g.
   `PRIMARY KEY (id, timestamp)`, `UNIQUE (alert_id, timestamp)`. A partitioned table cannot carry a
   unique constraint that omits its key, so a single-column PK forces a semantic change and an index
   rebuild at conversion time. Declaring the composite form up front makes conversion index-free.
3. **No table FKs to a member on a single column.** Referencing tables carry
   `(alert_id, alert_timestamp)` or a bare UUID (control-plane ADR §2.10 already establishes bare UUIDs across
   the plane boundary). Note the PoC's stated reason is imprecise: since PG12 a foreign key *may*
   reference a partitioned table — what it may not do is reference a unique constraint that omits the
   partition key.
4. **Retention filters on the member's time column**, matching the register. The per-table
   `TableRetention { table, time_column }` mapping introduced by ISSUE-296 is the right shape; the
   register is its source of truth.
5. **Read APIs take a bounded time range as part of the request contract**, not as an optional
   filter. Unbounded `ORDER BY <time> DESC LIMIT n` over a whole table is the query shape that
   defeats pruning after conversion, and it is also the shape that made the PoC's 5-second dashboard
   poll run an unbounded `count(*)`.

### 2.4 Conversion triggers

Any one of these, observed on any single tenant, moves that member to partitioned:

| Trigger | Threshold |
|---|---|
| Row count | > 50M rows in one tenant's copy of the table |
| Size | > 100 GB total relation size, or the hot working set no longer fits `shared_buffers` |
| Retention horizon | configured beyond 730 days *and* annual growth above ~5M rows — `alerts_days` accepts up to 3650, so this can arrive as a config change with no code review |
| Retention cost | a retention run exceeding ~5 minutes, or producing measurable write stalls or bloat that `VACUUM` does not recover between runs |

Crossing a threshold is a trigger to convert, not licence to convert every member.

### 2.5 Prescribed mechanism when a trigger fires

Each clause rules out an alternative, which is why it is a decision rather than a technique.

- **Convert by `ATTACH`**, adopting the existing table as the historical partition behind a proven
  `CHECK` constraint (Appendix A). Cost scales with index count, not row count. This rules out
  build-new-and-backfill, which needs ~2× the table in free disk — a live constraint on an appliance
  sized at 100 GB.
- **Manage with `pg_partman` + `pg_cron`**, small `premake` (4–6). This rules out both mechanisms the
  PoC used: DDL `DO` blocks enumerating years, and an application-lifespan roller. The window rolls
  inside the database, so no service owns it and retiring a service cannot re-arm ISSUE-212.
- **Interval equals expiry granularity.** At 90-day retention that is weekly (~13 live partitions),
  not monthly. Target tens of live partitions, never hundreds.
- **Expire by `DETACH CONCURRENTLY` then `DROP`.** This rules out row-wise `DELETE` for partitioned
  members, and is the specific mistake §1.1 documents. Detach-then-drop also leaves room to archive a
  partition before dropping it.

### 2.6 Retention executes inside the data plane, per tenant

Under the control-plane ADR §2.2, retention is per-tenant-database work. The PoC's approach — `liarbirdctl`
shelling out `kubectl exec … psql` against the CNPG primary — is appliance-shaped and does not
generalise to N tenant databases. Retention in the rebuild is a scheduled job in the data plane,
parameterised by the register, reading each tenant's configured periods.

---

## 3. Considered and rejected

**3.1 Partition by month up front (what the PoC did).** Rejected on two years of evidence: one P0
(ISSUE-212), two retention defects (ISSUE-280/296), a staging migration disabled for a schema
mismatch, a stripped FK, ~1,600 index relations per tenant to hold ~4 months of data, and pruning
that fails on most read paths. None of that was caused by monthly granularity as such; all of it was
caused by partitioning before there was a reason to.

**3.2 Decide per table, ad hoc.** The status quo. It produced `alerts` partitioned and `audit_logs`
— append-only, 4× the retention — not, with the only written deliberation being the one that said no.
The failure mode is not a wrong call; it is a call nobody can find, evaluate, or revisit.

**3.3 TimescaleDB hypertables from the start.** Compression (typically 10–20× on this data shape) is
genuinely the strongest lever for security telemetry on a 100 GB appliance, and hypertables subsume
chunk management and retention policies. Deferred, not dismissed: it adds an extension to every
tenant database and to the appliance's CNPG image for a benefit that only materialises at volumes
§2.2 assumes are absent, and compression is TSL-licensed rather than Apache-2 — a decision that needs
its own ADR rather than a clause in this one.

**3.4 Scope this ADR on data volume ("managing high-volume tables").** Rejected: an entry condition
of "high volume" only binds tables someone has already judged to be large, which is precisely the
unrecorded judgement §1.1 documents. Lifecycle is checkable from the DDL; volume is not.

**3.5 Partition `audit_logs` on its merits.** It is the best-qualified member — truly append-only,
`NOT NULL` time column already, longest retention. Rejected for the same reason as the rest: no
measurement suggests it is near any §2.4 threshold. It is named here so that the next reader can see
the candidate was considered rather than overlooked.

**3.6 Persist normalized telemetry and partition that.** No such table exists (§1.2) and none is
proposed. Including it as an open question would import the same unexamined assumption this ADR
exists to prevent. If the rebuild later decides to persist the event stream, that is a new decision
that supersedes this one.

---

## 4. Consequences

### Positive

- The partitioning decision becomes reversible in one direction that is cheap (§2.5) and is never
  taken in the direction that is expensive.
- `alerts` regains a single-column primary key and can be an FK target, removing the
  application-level integrity checks the PoC's comments call for.
- Retention becomes a `DELETE` filtered on an indexed time column — correct, boring, and identical in
  shape across all four members.
- No service owns a partition roller, so service retirement cannot re-arm ISSUE-212.
- The register makes the class enumerable, so the next table that qualifies inherits a decision
  instead of prompting a fresh guess.

### Negative

- If the §2.2 volume assumption is wrong for a large tenant, that tenant hits a conversion project
  rather than having been partitioned from the start. §2.3 caps the cost at an `ATTACH` plus
  `pg_partman` adoption; §7 exists to catch it before a customer does.
- Conversion is per-tenant fleet work (control-plane ADR §2.13), so the trigger fires per tenant and the
  response is not a single migration.
- The migration runner wraps each file in one transaction holding an advisory lock
  (`run_migrations.py:308-321`), and `CREATE INDEX CONCURRENTLY` / `DETACH PARTITION CONCURRENTLY`
  cannot run inside a transaction block. A future conversion needs either a non-transactional runner
  mode or out-of-band steps. Worth knowing before it is needed, not worth solving now.

### Neutral / already true

- Postgres is 16 in local compose and 15 in the appliance chart. Both support everything §2.5 and
  Appendix A require (`ATTACH` skip-scan since 12, `DETACH CONCURRENTLY` since 14).
- Nothing FKs to `alerts` today, so invariant 3 is satisfied on arrival rather than needing a change.

---

## 5. Constraints this decision imposes

1. A new table meeting §2.1 must be added to the register in the same change that creates it.
2. A schema change may not introduce a single-column FK to a register member, nor a unique constraint
   on a member that omits its time column, without amending this ADR.
3. A read endpoint over a register member must accept and default a bounded time range.
4. Retention configuration limits must stay consistent with §2.4 — raising a member's maximum
   retention above 730 days is a change that touches this ADR.

---

## 6. Invariants to enforce in CI

Each is a test, not a convention.

1. **Every register member's time column is `NOT NULL`** in the live schema.
2. **Every register member's primary key and natural-key unique constraints include its time
   column.** This is invariant 2 of §2.3 and the single check that preserves cheap conversion.
3. **No foreign key references a register member on a single column.**
4. **The retention job's table/time-column mapping matches the register exactly** — both directions,
   so a renamed table or column fails loudly. The PoC shipped a retention `DELETE` against a
   `responses` table that did not exist and an `audit_logs.created_at` column that did not exist;
   both were silent no-ops.
5. **Retention SQL filters on the member's time column** and, for any partitioned member, the outer
   statement carries that predicate directly rather than delegating it to a subquery.
6. **The register enumerates every table meeting §2.1** — a schema introspection test that flags any
   table with a `NOT NULL` timestamp column and a retention reference that is absent from the
   register.

---

## 7. Validation gate

**Agent-reported detections are the only remaining source of `alerts` rows.** The retirement of the
Analysis service removes rareness/anomaly scoring, threat-intel and GeoIP enrichment, and the parser
pipeline for every non-agent log source; agent telemetry becomes the only ingest path
(`docs/investigations/2026-08-05-endpointmgr-only-retirement-audit.md`, assumptions A5 and A8, §2.3).
So the PoC's ingest-coupled admission path — `should_trigger_enrichment` admitting any event with a
rareness field `> 0.95` (`main.py:842`), a *relative* threshold whose output rate tracks ingest volume
and field cardinality rather than attacker activity — does not exist in the rebuild. EndpointMgr's
`alert_processor` writing dedup-merged agent detections is the whole of it.

That does not rest on the audit's assumptions alone. Rareness is produced in exactly one place:
`finalize_event` (`analysis/parsers.py:78-84`) calls `track_event_fields` then
`calculate_event_rareness` at *parse* time, scoring each event against Redis frequency counters it has
just incremented (`analysis/rareness_service.py:434`). No parser definition maps a rareness field from
an agent payload, and `src/endpointmgr/` contains zero references to rareness. There is no code path
by which the scores survive the retirement of the parser pipeline, independently of whether
enrichment is retired as a capability.

That removes the mechanism by which `alerts` could have grown at telemetry rates, and is the main
reason §2.2's assumption is expected to hold rather than merely hoped to. The gate is therefore
narrower than a volume study:

- decoy-trigger alerts/day per tenant on staging or a design partner, and the same figure normalised
  per agent;
- projected steady-state rows at each member's default retention, per tenant, from that rate;
- the same for `audit_logs`, whose rate is driven by operator and agent activity rather than
  detections and which retains 4× as long.

If per-agent detection rates put any member within an order of magnitude of a §2.4 threshold at
plausible fleet sizes, that member starts partitioned rather than converting later. Either way the
decision is then made on a number — which is the one thing §1.1 shows was never true before.

---

## 8. References

- [Control plane / tenant plane separation](control-plane-tenant-plane-separation.md) — §2.2 (storage
  topology), §2.10 (cross-plane FKs), §2.13 (tenant-lineage migrations as fleet
  operations)
- `database/init/01-schema.sql:209-210,300` — the partitioning decision and its only stated rationale
- `appliance/liarbirdctl/src/retention/postgres.rs:58,162-166` — `DELETE`-based retention and its
  non-pruning predicate
- `EagerBeaver/src/shared/partition_management.py`, `src/analysis/main.py:150` — the roller and its
  sole caller
- `EagerBeaver/src/analysis/main.py:507,793-852` — what admits an event to `alerts`
- `EagerBeaver/src/analysis/parsers.py:78-84`, `rareness_service.py:434` — where rareness scores are
  produced, and the Redis counters they are measured against
- AgileFramework `docs/archive/m2/stories/story-m2-6.6.md:514` — `audit_logs` partitioning declined
- AgileFramework `docs/stories/story-m3-1.3.md:482-509`, `story-m3-10.1.md:90,230` — costs incurred
- AgileFramework `docs/known-issues-archive.md` — ISSUE-212 (partition window exhaustion),
  ISSUE-280/296 (retention predicate defects)

---

## Appendix A: Conversion sketch

Illustrative, not a runbook — included so the §2.2 claim that conversion is cheap is auditable rather
than asserted. The operational procedure — lock timeouts, disk headroom, per-tenant sequencing,
verification, rollback — belongs in `liarbird-docs/internal/operations/` and should be written when a
§2.4 trigger fires, against the Postgres version in production at that time.

Executed against PostgreSQL 16.14 on 2026-08-05, on a 2M-row / 398 MB table carrying the §2.3
composite PK and unique constraint, two secondary indexes, and a `BEFORE UPDATE` trigger:

| Step | Measured |
|---|---|
| `ADD CONSTRAINT … NOT VALID` | 0.6 ms |
| `VALIDATE CONSTRAINT` (online, `SHARE UPDATE EXCLUSIVE`) | 75 ms |
| Whole step-2 transaction, `RENAME` → `CREATE` → `ATTACH` | **2.0 ms**, of which `ATTACH` 0.4 ms |
| Same `ATTACH` with no proven `CHECK` (validation scan runs) | 71 ms, and scales with row count |
| Rows copied | zero — `pg_relation_size('alerts')` on the new parent is 0 |

`pg_partman` calls are 5.x signatures, confirmed against the extension's own documentation:
`create_parent(p_parent_table, p_control, p_interval, p_type DEFAULT 'range', …, p_premake DEFAULT 4, …)`,
and `retention_keep_table = false` is what makes expiry a `DROP` rather than a bare `DETACH`.

```sql
-- 1. Prove the historical bound online. NOT VALID skips the initial scan;
--    VALIDATE takes only SHARE UPDATE EXCLUSIVE, so writes continue.
ALTER TABLE alerts ADD CONSTRAINT alerts_hist_bound
  CHECK (timestamp >= '2026-01-01' AND timestamp < '2027-01-01') NOT VALID;
ALTER TABLE alerts VALIDATE CONSTRAINT alerts_hist_bound;

-- 2. Swap in a partitioned parent and adopt the old table. The proven CHECK lets
--    ATTACH skip validation, so the exclusive lock is held for well under a second
--    regardless of table size, and no row is copied.
BEGIN;
ALTER TABLE alerts RENAME TO alerts_hist;
CREATE TABLE alerts (LIKE alerts_hist INCLUDING ALL) PARTITION BY RANGE (timestamp);
-- INCLUDING ALL copies CONSTRAINTS, so the parent inherits the historical CHECK
-- and would reject every row outside 2026. Drop it from the parent only.
ALTER TABLE alerts DROP CONSTRAINT alerts_hist_bound;
ALTER TABLE alerts ATTACH PARTITION alerts_hist
  FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
COMMIT;

-- LIKE does not copy triggers. Any member with one — audit_logs carries the
-- append-only trigger from migration 069 — needs it recreated on the parent.
-- LIKE does copy indexes, but names collide with the originals still on the old
-- table, so the parent's arrive suffixed (alerts_pkey1). Rename for clarity.

-- 3. Hand the window to pg_partman, then retention becomes DROP rather than DELETE.
SELECT partman.create_parent('public.alerts', 'timestamp', '1 week',
                             p_premake => 4);
UPDATE partman.part_config
   SET retention = '90 days', retention_keep_table = false
 WHERE parent_table = 'public.alerts';
```

Step 1 is only needed when converting a populated table. A member that starts partitioned skips to
step 3.
