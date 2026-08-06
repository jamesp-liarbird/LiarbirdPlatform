# Dashboard carry-over

The dashboard is carried across rather than rebuilt. At roughly 104,000 lines of Next.js and
TypeScript it is the largest single component in the estate, and most of it serves capabilities that
are unaffected by the descope.

This document names the parts that are **known** to need rebuilding. Everything else is copied as-is
and validated by a testing pass afterwards, with whatever that pass finds raised as bugs. The
distinction matters because the dashboard is already partly broken: `analysis` and `responder` have
been taken out of action ahead of it, so surfaces backed by them are dead in the running system
today. Copying as-is copies that state deliberately — it is a starting point to triage from, not a
claim that the carried code works.

## 1. What holds across the whole carry

- **The stack is unchanged.** Next.js and TypeScript, as today.
- **API paths do not change.** The backend keeps the `/api/v1/endpointmgr/…` prefix it has now. The
  agent wire contract pins those paths permanently, so re-cutting the dashboard's along module lines
  would leave one server answering two prefixes. The service layer therefore ports untouched, and
  the modular structure behind it ([`../adr/architecture-style.md`](../adr/architecture-style.md) §4)
  is invisible to the browser.
- **The dashboard publishes nothing.** Its `src/app/api/**/route.ts` handlers and `next.config.js`
  rewrites are same-origin proxies for its own frontend. There is no external consumer, so a change
  on either side ships as one commit.
- **No structural conditions are imposed on carried code.** The rules in
  [`backend-carry-over-rules.md`](backend-carry-over-rules.md) govern the backend and none of them
  apply here. This is a deliberate acceptance of drift for now: the first substantial rework is the
  point to revisit it, not the copy.

## 2. Rebuild — the known set

Four surfaces. Everything else is §3 or §4.

### 2.1 The landing page becomes `/alerts`

The current landing page is the Command Centre at `/`, and its content is entirely the attack-chain
graph — `src/app/page.tsx` renders `components/CommandCentre/` including `GraphView3D.tsx` and
nothing else. The graph leaves with the descope
([`../decision-backlog.md`](../decision-backlog.md) §3), so `/` has no content.

`/alerts` takes its place. It costs nothing, it is the surface an operator looks at most, and every
human role carries `ALERT_VIEW` (`shared/middleware/rbac.py`), so no role lands on a permission
error. A purpose-built overview remains reachable later; this is the default that avoids building
one now.

### 2.2 `/countermeasures` — rebuild against the execution ledgers

The page is the pending-response queue: the list, the approve/reject dialogs, the ramp stages, and an
executed history that reads the same table. Both halves read approval fields — `reviewed_by`,
`response_status` — so this is a rewrite, not a repoint.

It rebuilds as a dispatch form, live responses from `response_actions`, and history from `commands`,
per [`capabilities/response-lifecycle.md`](capabilities/response-lifecycle.md) §3.3. `/response-queue`
is a redirect into this page and follows it.

This is the one surface where "remove what the descope broke" gives the wrong answer. Its backing
table loses its only writer, but the capability survives with a different producer
([`../adr/operator-initiated-endpoint-responses.md`](../adr/operator-initiated-endpoint-responses.md)).

### 2.3 `/alerts` — remove the enrichment surfaces

Structurally fine and endpointmgr-backed, but it renders state that no longer exists: the "Enriching"
indicator and its `isActivelyEnriching()` time-window heuristic, the Enrichment
Complete/In-progress/N/A field, and the MITRE tactic column. All go, per
[`capabilities/alert-ingestion.md`](capabilities/alert-ingestion.md) §3.4 and §4.

The indicator is worth removing rather than leaving: with no enricher it shows "In progress…" for its
window and then "N/A" permanently, a progress display that can never complete. §2.1 makes this the
landing page, so it is also the surface with the most removals.

### 2.4 `/integrations` — split, don't delete

The SIEM configuration UI lives here rather than at `/settings/siem`. The integration tiles and the
licence surface leave with `analysis`; `SiemConfigurationModal`, `siemSettingsService` and the
per-connector delivery-status read survive.

`src/app/settings/siem/page.tsx` is a short redirect *into* `/integrations`. Rehoming the SIEM tiles
there and inverting the redirect reuses a page shell that already exists. Where the forwarder itself
lands is [`../decision-backlog.md`](../decision-backlog.md) decision 6.

## 3. Remove

Surfaces whose backing service is not carried across. These are deletions, not rebuilds.

| Removed | Backed by |
|---|---|
| Command Centre `/`, `components/CommandCentre/`, the `/api/graph/*` routes | `analysis` → the graph |
| Anomaly Detection, Selection Logic, Data Sanitisation | `analysis` |
| Agent Dynamics, Agent Config, Agent Tuning, and the unlisted `/responder/*` routes | `responder` |
| Forwarding sessions, and the `/settings/forwarding/*` configuration routes | `forwardingrelay` |
| `services/{responder,eagerbeaver,commandCentre}.ts` and their tests | as above |
| `next.config.js` rewrites — 3 responder, 5 analysis | as above |
| `/responses` and its redirect | Already unreachable: `next.config.js` permanently redirects it to `/countermeasures` |

The audit's §5.1, §5.2 and §5.7 enumerate these route by route.

**Net effect on the nav:** Main keeps DETECT, DISRUPT, DEPLOY and PERFORMANCE, but loses its landing
page (§2.1), the Integrations child (§2.4), Data Sanitisation, and DISRUPT's Forwarding child.
Appliance and Settings are untouched. Admin loses five of its six items, leaving only System
Settings — worth folding into Settings rather than keeping a one-item group.

## 4. Copy as-is

Everything not named in §2 or §3, including Decoys and Deployment Limits, the DEPLOY group, the
PERFORMANCE surface, all ten Appliance items, all four Settings items, and the login, logout,
onboarding, terms, legal and help routes.

Correctness of the carried set is established by the testing pass, not by inspection here. Anything
that pass finds is a bug ticket against carried code, which is the intended outcome rather than a
failure of the carry.

## 5. Related

- [`backend-carry-over-rules.md`](backend-carry-over-rules.md) — the backend rules, none of which
  apply here (§1)
- [`capabilities/response-lifecycle.md`](capabilities/response-lifecycle.md) — what `/countermeasures`
  rebuilds against
- [`capabilities/alert-ingestion.md`](capabilities/alert-ingestion.md) — what `/alerts` stops
  rendering
- [`../adr/architecture-style.md`](../adr/architecture-style.md) — §4.1 records that the dashboard
  stays a separate deployable under every option
- [`../investigations/2026-08-05-endpointmgr-only-retirement-audit.md`](../investigations/2026-08-05-endpointmgr-only-retirement-audit.md)
  — §5, the route-by-route sort this document rests on
