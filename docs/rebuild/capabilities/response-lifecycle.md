# Response lifecycle

An operator disrupts an endpoint; the endpoint executes the disruption and reports on it. Removing
automated selection removes what decided a response should happen and nothing else — dispatch,
delivery, acknowledgement and the execution ledgers are already `endpointmgr`'s, so most of this
capability carries across working.

What follows is the delta: what holds, what the wire contract fixes, what changes, and what is
deliberately absent. The decision behind it is
[`../../adr/operator-initiated-endpoint-responses.md`](../../adr/operator-initiated-endpoint-responses.md);
the cross-cutting structural changes are in
[`../backend-carry-over-rules.md`](../backend-carry-over-rules.md), and §6 notes which of them land
hardest here.

## 1. What already holds

Carry these across on their behaviour, subject to §6:

- **The dispatch route** (`endpointmgr/api/response.py:123`) — validates that a target field is
  present, enforces that the caller's tenant owns the agent, checks agent readiness and command
  conflicts, audits rejections, and writes a command. It is the ADR's initiation path and needs an
  operator surface, not a rewrite.
- **Command delivery and the payload transform** (`endpointmgr/api/agent_registration.py:2366`) —
  maps the operator-facing action to the opaque wire type, flattens the target, merges parameters,
  and carries `process_lineage` and `alert_id` through for agent-side re-evaluation.
- **Acknowledgement, stop, stop-all and cancel** (`api/response.py:238,352,419`) — the agent-facing
  acknowledge path and the operator-facing halt paths.
- **The two execution ledgers.** `commands` is the delivery record, read by the history surface via
  `CommandRepository` (`:479`). `response_actions` is the execution record, read by the live and log
  surfaces via `ResponseActionRepository` (`:764,788`). They are distinct and both survive.
- **Per-asset response mode resolution** (`endpointmgr/api/manifests.py:121-201`) — resolves each
  deployed honeypot's mode from its detection profile at manifest build time. This is what the agent
  reads to decide which path runs.

## 2. Fixed by the wire contract

Agents in the field constrain these; none may change.

- `POST /api/v1/endpointmgr/agent/responses` and
  `POST /api/v1/endpointmgr/response/command/{id}/acknowledge`.
- Commands are delivered on the heartbeat, not pushed.
- **A response command must carry `ramp_schedule`, `r_type` or `response_type`**, or the agent
  rejects it before execution (`LiarbirdAgent/src/commands.rs`).
- **`r_type` is exactly `action_a`, `action_b` or `action_c`.** There is no legacy-name fallback, by
  design — an unmapped value is an unknown-response-type error on the endpoint.
- **The target is one of `target_ip`, `target_pid` or `target_process`.** Absent all three, the agent
  errors.
- Cancellation accepts `target_command_id` or `response_id`.
- The manifest's `r_cfg` shape, and the per-asset response mode the agent reads from honeypot data.

## 3. What changes

### 3.1 An operator initiates, and `hitl` terminates

A response is initiated by an operator dispatching against §1's route. The per-asset mode keeps its
three values and `hitl` gains a terminating behaviour: the agent holds, and the alert surfaces as
something an operator can act on. `immediate` still responds autonomously and `alert_only` still
suppresses.

This closes the gap the ADR's §1.5 names — an asset set to `hitl` currently defers to an approver
that does not exist, producing neither a response nor a record.

### 3.2 The approval queue does not come across

The pending-response table, its list, detail, approve, reject, cancel and history routes, the expiry
sweep that claims its rows, and the queue service behind them
(`shared/services/response_queue.py`, `endpointmgr/api/response_queue.py`,
`endpointmgr/tasks/pending_response_expiry.py`). Approval was an interlock on machine-initiated
action; ADR §4 records why it does not survive the machine.

Attribution moves with it. Where the queue recorded `reviewed_by` separately from the request, the
command's creating actor is the whole record.

### 3.3 The operator surface is rebuilt, not repointed

Both halves of the current disrupt page read approval fields — the pending list and the executed
history alike (`dashboard/src/app/countermeasures/page.tsx:123-124,225-231`). Against the ledgers in
§1 it is a rewrite: a dispatch form, live responses from `response_actions`, history from `commands`.

### 3.4 Reads do not write

The live-responses handler runs `expire_stale_actions()` and commits inside a `GET` (`:770-774`), so
a read mutates state and its effect depends on who happens to be looking. Expiry is a scheduler
concern ([`../backend-carry-over-rules.md`](../backend-carry-over-rules.md) rule 5); the read surface
reads.

### 3.5 Decide whether a commanded response carries a schedule

The payload transform emits no `ramp_schedule`, so commanded responses run the agent's built-in
ladder — 10% for 60s, 30% for 120s, 50% for 180s, then 70% indefinitely
(`LiarbirdAgent/src/response/schedule.rs`). That is a defensible default and it is currently
undocumented behaviour rather than a choice.

Either the transform carries a schedule and the dispatch surface exposes it, or the default is
written down as the commanded profile. Note what the default costs: it is identical on every
endpoint, where the randomisation that makes a response hard to fingerprint belongs to the
autonomous path alone (ADR §1.2).

## 4. Deliberately absent

| Absent | Why |
|---|---|
| Automated response selection | Deferred with its service. It returns as a module with its own approval design and storage (ADR §4.4), not by refilling a queue kept for it |
| `pending_responses` and its lifecycle | §3.2 |
| Second-person authorisation | Nothing stands between an operator and a disruptive action but permissions and the audit trail. A separation-of-duties requirement is built explicitly on the dispatch path, not by reinstating the queue (ADR §4.2) |
| The optional command-dispatcher hook on approval | A protocol seam whose only implementer was the deferred service's adapter |

## 5. Handoffs

- **Manifest envelope (decision 7).** `permitted_actions` bounds what the autonomous path may do and
  currently fails open — an absent action key reads as permitted. The bounds that path randomises
  within have a wire name, `ir_cfg`, that the server never sends. Both are that decision's, and ADR
  §4.4 requires they be settled rather than inherited.
- **The dashboard.** §3.3's rebuild is one of the four named in
  [`../dashboard-carry-over.md`](../dashboard-carry-over.md) §2.2, and the only surface there whose
  backing table loses its writer while the capability survives.
- **Alert ingestion.** A response references the alert that provoked it, and the transform carries
  `alert_id` to the endpoint. The link is one-way: alerts do not depend on responses.

## 6. Carry-over rules that land hardest here

- **Rule 5 (background work).** The expiry sweep leaves with the queue (§3.2), and it is the estate's
  only `FOR UPDATE SKIP LOCKED` claiming precedent. Anything later needing a replicated worker
  rebuilds that shape from the rule rather than inheriting it, and `tasks/scheduled_cleanup.py` is
  named there as the shape not to copy. §3.4 is the other half of this rule: the expiry that runs
  inside a read belongs to the `scheduler` role.
- **Rule 1 (connections).** The dispatch route takes two sessions — the platform database for the
  cross-tenant agent lookup and audit writes, the tenant database for the command write
  (`api/response.py:126-129`). Both are acquired as route dependencies, so the seam is already at a
  chokepoint; the question is which plane each write belongs to, answered by
  [`../../adr/control-plane-tenant-plane-separation.md`](../../adr/control-plane-tenant-plane-separation.md).
- **Rule 2 (repositories).** Mixed. History and the live surfaces already go through
  `CommandRepository` and `ResponseActionRepository`; the dispatch path builds its own queries
  alongside `CommandExecutor`. Those move into the repository.
- **Rule 4 (table ownership).** `commands` and `response_actions` move out of the shared layer and
  are declared by `endpoints`, in the tenant plane.
- **Rule 6 (grouping).** The surviving routes are spread across two modules that split along the
  approval boundary, which stops existing. They land as one domain group.

## 7. Related

- [`../../adr/operator-initiated-endpoint-responses.md`](../../adr/operator-initiated-endpoint-responses.md)
  — the decision this brief implements, including the two execution paths, the per-asset mode gate
  and §4.3's assertions
- [`../backend-carry-over-rules.md`](../backend-carry-over-rules.md) — the structural changes that
  apply to all carried backend code
- [`../../adr/architecture-style.md`](../../adr/architecture-style.md) — `endpoints` owns this
  capability, and §4.3's requirement that a capability name its owning module and public surface
- [`alert-ingestion.md`](alert-ingestion.md) — the alert a response references, and the `ai_*`
  columns that left with automated analysis
- [`../../investigations/2026-08-05-endpointmgr-only-retirement-audit.md`](../../investigations/2026-08-05-endpointmgr-only-retirement-audit.md)
  — the coupling map, §5.4 for the disrupt surface
