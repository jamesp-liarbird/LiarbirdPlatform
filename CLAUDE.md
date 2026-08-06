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

## Investigations

`docs/investigations/` holds dated evidence documents that ADRs cite — assessments and audits of the
proof-of-concept, named `YYYY-MM-DD-topic.md`. They are records, not decisions: never edited to
match a later decision. An ADR cites primary artifacts for its own claims so that its argument
stands without them.

## Architecture decision records

Live in `docs/adr/` and follow [MADR 4.0](https://github.com/adr/madr). Start one by copying
`docs/adr/_template.md`, or `_template-minimal.md` for a decision that doesn't block others.

Section-level guidance lives in the template itself, including the local deviations from upstream
MADR — all marked `LOCAL`, so the template can be re-synced against a future MADR release.

Numbering is unresolved (see the backlog), so ADRs are named and cross-referenced by slug for now
and get a number when the estate decision lands. The org's existing series — plain `ADR-001…005` in
AgileFramework, plus `ADR-MT-*`, `ADR-M3-*` and `ADR-VM-*` embedded as subsections inside
`AgileFramework/docs/architecture-m3-*.md` — are foreign to this repo and are referenced by ID.
The frontmatter's `related` and `supersedes` carry those slugs and IDs as bare identifiers; a
supersession that holds only in part is stated in More Information rather than qualified in place.

The two ADRs carried over from the PoC repo predate this format. They keep their own structure and
read backwards, as diffs against the running system. Treat them as content to build on, not as
examples to copy — new ADRs follow the template.

## Standing constraints

Every ADR is tested against these. They are settled — a decision that breaks one needs to say so
explicitly and argue for it.

- **Two deployment shapes.** SaaS multi-tenant with isolated per-tenant databases, and single-tenant
  self-hosted. Both are first-class; neither is a degraded mode of the other.
  [`docs/deployment-shapes.md`](docs/deployment-shapes.md) records the full set of shapes and which
  are settled.
- **The agent wire contract.** Deployed agents already speak ~25 `/api/v1/endpointmgr/…` paths
  (register, bootstrap, heartbeat, refresh, alerts, responses, manifest, updates, uninstall). The
  server's API surface is constrained by what is already in the field.
- **Data stays in Australia/New Zealand.** Customer data at rest and in backups is held in AU/NZ
  regions. Serving a tenant under another jurisdiction's residency requirement is an open question,
  not a configuration change.
- **Scope is endpointmgr first.** The `analysis`, `forwardingrelay` and `responder` services are not
  being brought across. Where endpointmgr reaches them today, that seam is a decision to be made,
  not a dependency to be recreated.

## Decision principles

How to choose, where the constraints above leave room. Unlike them, these are heuristics — an ADR
that departs from one should say why, but departing is not a defect.

- **The simplest thing that isn't a corner.** Take the least complex option that leaves a more
  complex one reachable, and name what would trigger the upgrade. Without a trigger, "later if we
  need to" has no arrival condition.
- **It applies to the design, not the guardrails.** Deferring structure is cheap. Deferring the
  checks that keep structure honest is what makes the upgrade expensive, because what accumulates
  in the meantime is undetected drift, not complexity.
- **Defer a boundary only if moving it later stays local.** What makes relocation expensive is the
  number of places that restate the boundary's position — not whether it lives in code, the schema
  or a wire format. One decided at a single chokepoint is deferrable; one restated at every call
  site is cheap now and expensive at any later point. Centralising the decision buys deferability.

## Writing

House style for editing existing documents is in `../CLAUDE.md`. The rule that matters most here:
write for the steady-state reader, not the transition. A future reader never saw the PoC and should
not need to.
