# LiarbirdPlatform

The rebuild of the Liarbird platform. **This repo currently holds documents, not code.** The first
phase is architecture and decision records; it becomes the working code repository once the
decision backlog in `docs/decision-backlog.md` is cleared.

`LiarbirdServer` is the proof-of-concept being rebuilt from. It is a source of *evidence*, never of
*precedent* — "the PoC does X" is an observation to be argued from, not a reason to do X. Do not
copy code from it into this repo.

## Reference conventions

- Bare code paths (`database/init/01-schema.sql`, `EagerBeaver/src/shared/tenant/`) refer to
  **LiarbirdServer**.
- Paths beginning with another repo name (`AgileFramework/docs/…`, `liarbird-docs/…`,
  `liarbird-helm/…`) refer to that repo.
- `ISSUE-NNN` and `story-mN-x.y` are AgileFramework's tracker.
- Line numbers are provenance, not durable references. Expect them to rot.

## Architecture decision records

Live in `docs/adr/` and follow [MADR 4.0](https://github.com/adr/madr). Start one by copying
`docs/adr/_template.md`, or `_template-minimal.md` for a decision that doesn't block others.

House conventions on top of MADR:

- **`status` names its promotion trigger**, never a bare `proposed` — `proposed — accepted once the
  Phase 1 slice confirms X`. A status with no named event never moves.
- **Ask whether the decision behaves differently across the two deployment shapes.** Usually it
  doesn't and there's nothing to write. When it does, say where the divergence is absorbed — a seam,
  a provider, a config object — so it lives in one place. Minor cases belong in Decision Drivers,
  substantive ones get their own subsection under Decision Outcome.
- **`Confirmation` names tests where there is an enforceable surface.** "Each is a test, not a
  convention" — this is what stops a design eroding after it's accepted.
- **`Constraints imposed`** (a local addition) is for obligations on *future* work, as distinct from
  Consequences, which is what follows from the decision itself. Optional.
- **PoC evidence is restated as a finding**, not as a diff against a system the reader doesn't know.
  "Range-partitioning a 90-day-retention alerts table costs ~1,600 index relations per tenant and
  delivers neither bulk expiry nor pruning" — not "the PoC partitioned alerts at `01-schema.sql:300`
  and never used `DROP PARTITION`." Keep the citation for provenance; lead with the finding.

Numbering is unresolved (see the backlog), so ADRs are named and cross-referenced by slug for now
and get a number when the estate decision lands. The org's existing series — plain `ADR-001…005` in
AgileFramework, plus `ADR-MT-*`, `ADR-M3-*` and `ADR-VM-*` embedded as subsections inside
`AgileFramework/docs/architecture-m3-*.md` — are foreign to this repo and are referenced by ID.

The two ADRs carried over from the PoC repo predate this format. They keep their own structure and
read backwards, as diffs against the running system. Treat them as content to build on, not as
examples to copy — new ADRs follow the template.

## Standing constraints

Every ADR is tested against these. They are settled — a decision that breaks one needs to say so
explicitly and argue for it.

- **Two deployment shapes.** SaaS multi-tenant with isolated per-tenant databases, and single-tenant
  self-hosted. Both are first-class; neither is a degraded mode of the other.
- **The agent wire contract.** Deployed agents already speak ~25 `/api/v1/endpointmgr/…` paths
  (register, bootstrap, heartbeat, refresh, alerts, responses, manifest, updates, uninstall). The
  server's API surface is constrained by what is already in the field.
- **Scope is endpointmgr first.** The `analysis`, `forwardingrelay` and `responder` services are not
  being brought across. Where endpointmgr reaches them today, that seam is a decision to be made,
  not a dependency to be recreated.

## Writing

House style for editing existing documents is in `../CLAUDE.md`. The rule that matters most here:
write for the steady-state reader, not the transition. A future reader never saw the PoC and should
not need to.
