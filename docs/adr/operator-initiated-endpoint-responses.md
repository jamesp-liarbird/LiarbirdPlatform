---
status: "proposed"
date: 2026-08-06
scope: the endpoints module's response lifecycle, the agent manifest's response envelope, and the
  operator-facing disrupt surface
related: architecture-style
---

# Operator-initiated responses without an approval queue

## 1. Context and Problem Statement

### 1.1 What must be settled

A response is a disruptive action taken on an endpoint — degrading its network, throttling or
suspending a process, isolating it. Something must decide that one should happen. Automated
selection is deferred along with the service that performed it, so the rebuild has to say what
initiates a response in its absence, or the capability arrives by default in a shape nobody chose.

This is not a question about where code lives. The queue machinery is already outside the deferred
service and already consumed by the surviving one; the deferred half is selection, and selection has
no replacement pending.

### 1.2 Responses execute on the endpoint, by two paths

Both paths run entirely on the agent, and the server's role differs sharply between them.

| Path | Initiated by | Schedule |
|---|---|---|
| Commanded ramping | A server command, delivered on the heartbeat | The schedule in the command, or a fixed default ladder |
| Autonomous response | The agent's own alert, evaluated locally | Randomised per execution within bounds compiled into the agent |

The default ladder is fixed at 10% for 60s, 30% for 120s, 50% for 180s, then 70% indefinitely
(`LiarbirdAgent/src/response/schedule.rs`), and it is what runs in practice: the server-side payload
transform emits no `ramp_schedule` field, so the agent falls through to it
(`endpointmgr/api/agent_registration.py`, `LiarbirdAgent/src/commands.rs`).

**The randomisation that keeps a response from being fingerprintable as a security control belongs
to the autonomous path only.** It draws delay, duration, intensity and ramp-up per execution from
`ImmediateResponseConfig` (`LiarbirdAgent/src/response/immediate.rs`), with packet-level jitter
beneath that. Commanded responses are identical on every endpoint.

Execution survives disconnection. Effect application continues when reporting fails, and progress
reports queue and flush on reconnect (`LiarbirdAgent/src/response/schedule.rs`). Server-side response
state is therefore a lagging report, not a control surface — the server cannot reliably observe a
ramp in progress even while connected, and cannot stop one by forgetting about it.

### 1.3 The manifest selects between the paths, per asset

The agent resolves the alert to a deployed honeypot and reads that asset's response mode
(`LiarbirdAgent/src/response/immediate.rs`). The server sets it, resolved per honeypot from the
detection profile at manifest build time (`endpointmgr/api/manifests.py`).

* `immediate` — the agent responds autonomously.
* `alert_only` — no response.
* `hitl` — **the agent returns without acting and waits for a server command.**
* Unmatched or unrecognised — treated as `alert_only`, so the gate fails closed.

### 1.4 The approval queue has no producer, and is the entire operator surface

`queue_response()` is called only from the deferred service's two selectors, and no route creates a
pending response (`shared/services/response_queue.py`, `endpointmgr/api/response_queue.py`). An
operator cannot initiate one today.

That queue is also everything the operator sees. The disrupt page is 1,051 lines of pending-response
list, approve/reject dialogs, ramp stages and executed history, and the response-queue route is a
redirect into it (`dashboard/src/app/countermeasures/page.tsx`). Both halves read the queue: the
executed view maps `reviewed_by` and `response_status`, which are approval fields, not command
fields.

A dispatch route exists and is fully built — target and agent-readiness validation, audited
rejections, permission-gated (`endpointmgr/api/response.py`). Nothing calls it. The dashboard client
issues only the `DELETE` form, to stop a running response.

### 1.5 Deciding nothing splits the fleet

Because the mode is per asset and the two gates fail in opposite directions, an absent server-side
initiator does not simply disable the capability:

* Assets set to `immediate` keep responding autonomously, with no dispatch path, no configuration
  surface and only lagging reports.
* Assets set to `hitl` produce no response and no server-side record that one was pending. The agent
  defers to an approver that does not exist.

The second is the more serious, because an absent response is indistinguishable from an absent
attack. This is what makes the decision a question about the capability rather than about a table.

### 1.6 The envelope is two fields, and the server sends one

`permitted_actions`, carried as `r_cfg`, is the agent's response envelope. It fails open: an absent
action key reads as permitted (`LiarbirdAgent/src/response/immediate.rs`).

The bounds the autonomous path randomises within have a wire name, `ir_cfg`
(`LiarbirdAgent/src/agent_config.rs`), and the server emits no such field
(`endpointmgr/api/manifests.py`). Autonomous responses therefore run on compiled-in defaults —
2–12s delay, 28–244min duration, 15%→80% intensity — which the server can permit or forbid but
cannot tune.

## 2. Decision Drivers

* **An approval gate is an interlock on machine-initiated action.** Its purpose is to put a human
  between an automated decision and a disruptive effect. With no automated decider, the same
  mechanism gates the human against themselves.
* **`hitl` must mean something.** It is a mode the server already assigns to assets and the agent
  already honours. Whatever is chosen has to give it a terminating behaviour.
* **The agent wire contract is fixed.** Deployed agents accept a bounded action vocabulary and
  deliver commands on the heartbeat. Nothing here may require an agent change.
* **Server-side state is a report.** A design that treats stored response rows as the control
  surface is describing something it does not control.
* **Autonomous response is already live.** It ships, it is server-gated per asset, and it is not
  waiting on a decision to begin.
* **Automated selection is deferred, not abandoned** (`architecture-style` §1.2). The interlock
  question returns with it, and should not be pre-answered by machinery kept idle in the meantime.

## 3. Considered Options

* **A. Operator-initiated dispatch, no approval queue.**
* **B. Operator-initiated approval — carry the queue across and add a producer.**
* **C. Endpoint-autonomous only — no server-side initiation.**
* **D. Defer the disrupt capability alongside automated selection.**

## 4. Decision Outcome

Chosen option: **A — operator-initiated dispatch, no approval queue.**

An operator approving a response the operator just proposed is a confirmation dialog implemented as
a table, a background expiry sweep and eight routes. The interlock has no machine to interlock
against, so what B preserves is the mechanism without the reason for it. A is also the only option
that gives `hitl` a terminating behaviour without inventing one: the mode already means "wait for a
server command", and A is what makes a server command arrive.

The initiation path is not new work. It exists, validated and permission-gated; what is missing is
an operator surface in front of it.

### 4.1 What follows

* **Initiation** is a dispatch call that writes a command. The agent picks it up on the heartbeat, as
  it does today.
* **The approval queue does not carry across** — the pending-response table, its list, detail,
  approve, reject, cancel and history routes, and the expiry sweep that claims its rows.
* **The disrupt surface is rebuilt against commands**, not approvals: a dispatch form, live
  responses, and history drawn from command records.
* **`hitl` means "an operator decides"** — the agent holds, and an alert on a `hitl` asset surfaces
  as something an operator can act on.
* **The autonomous path is retained**, gated per asset by response mode and bounded by
  `permitted_actions`.
* **The action vocabulary is the wire contract's**, not the dashboard's: the dispatch surface may
  offer only actions that map to the three opaque response types the agent accepts.

### 4.2 Consequences

**Good**

* `hitl` acquires a terminating behaviour, closing the silent gap in §1.5.
* Roughly 2,400 lines of queue machinery are not carried across, and one of the background workers
  that would otherwise need relocating under `carry-over-rules` rule 5 disappears with it.
* The operator gains an initiation path the product has never had, against a route already written.
* Response state stops being modelled as a control surface it cannot be (§1.2).

**Neutral**

* The two initiation paths compose rather than compete, which is what per-asset response mode
  already expresses. No new selector is introduced.
* Command records absorb what approval records held. Attribution moves from `reviewed_by` to the
  command's creating actor.

**Bad**

* The operator surface is a rewrite, not a repoint — both halves of the existing page read approval
  fields (§1.4). This is the bulk of the work and it is dashboard work.
* Nothing stands between an operator and a disruptive action but permissions and the audit trail.
  Where B would have required a second person, A requires that the first one is right.
* Response history loses its approval dimension. Anything that later wants "who authorised this,
  separately from who requested it" is a new schema change rather than a retained column.

### 4.3 Confirmation

1. **No module declares a pending-response table.** A model-inventory assertion over the declared
   entity set, run in CI alongside `architecture-style` §4.2 assertion 3.
2. **Every action the dispatch surface offers maps to an accepted wire type.** A test over the action
   enumeration and the wire mapping, so an unmapped action fails the build rather than reaching an
   agent that rejects it as an unknown response type.
3. **An alert on a `hitl`-mode asset produces an operator-actionable item.** An integration test
   asserting the §1.5 gap stays closed. This test does not exist yet and is written with the
   capability, not after it.
4. **The request-serving role registers no response-related background task.** Falls out of
   `architecture-style` §4.2 assertion 5; named here because the expiry sweep is the task most likely
   to be carried across out of habit.

Assertion 3 is the one that matters and the one with no existing surface to hang on. If the disrupt
capability ships without it, the failure it guards against is invisible by construction.

### 4.4 Constraints Imposed

* **The manifest envelope is a decision, not an inheritance.** `permitted_actions` currently fails
  open, and `ir_cfg` is never sent (§1.6). Whichever way those land, they are settled deliberately —
  a manifest that omits the envelope must not silently permit everything.
* **Automated selection reopens the interlock question when it returns.** Nothing here is reserved
  for it. It arrives with its own approval design and its own storage, and it may not assume this
  decision left a queue behind for it to fill.
* **Anything wanting a second-person check must build it.** Not by reinstating the queue, but as an
  explicit authorisation step on the dispatch path.

## 5. Pros and Cons of the Options

### B. Operator-initiated approval — carry the queue across and add a producer

Keeps the pending-response table, its routes and its expiry sweep, and adds the create route that
never existed so an operator can populate it.

**Pros**

* Preserves an approval record distinct from the dispatch record.
* The queue's row-claiming sweep is the one existing precedent for a worker that survives
  replication, so keeping it keeps a known-good pattern in the estate.

**Cons**

* The gate has nothing to gate. Proposer and approver are the same operator unless a separation is
  built on top, which is work B does not include.
* Carries a table, a worker and eight routes to express a state transition the operator performs in
  one action.
* The create route is new either way, so B does not avoid new API surface — it adds it *and* keeps
  the machinery.

### C. Endpoint-autonomous only — no server-side initiation

The manifest permits or forbids; the endpoint decides everything else.

**Pros**

* Smallest surface. No dispatch path, no operator surface, no queue.
* Matches where execution and randomisation genuinely live (§1.2).
* Already the shipping behaviour for `immediate` assets, so partly free.

**Cons**

* `hitl` becomes unreachable — an asset set to it responds never, silently (§1.5).
* Leaves the operator no way to act on a specific endpoint at a specific moment, which is the
  ordinary incident-response motion.
* Concentrates the entire control surface in a manifest field that currently fails open and a bounds
  field the server does not send (§1.6).

### D. Defer the disrupt capability alongside automated selection

**Pros**

* Smallest scope of all, and honest about automated selection being the thing that made the
  capability coherent.
* Nothing half-built is carried across.

**Cons**

* Does not stop responses. Autonomous execution is live and per-asset gated; deferring the server
  side leaves it running unobserved rather than turning it off (§1.5).
* Deferring it *properly* — so no endpoint responds — is itself work: the envelope must deny by
  default, which is a manifest change against a field that currently fails open.
* Removes a shipped product capability, which is a product decision this ADR is not the place to
  take.

## 6. More Information

The `hitl` finding in §1.5 is what changed this from a question about a table into a question about
the capability, and it is the part most worth re-checking if the agent's response gating changes:
the behaviour rests on the agent returning without acting in that mode, and on the per-asset mode
being server-assigned.

What would reopen this decision is automated selection returning. At that point there is a machine
initiator again, the interlock has something to interlock against, and §4.4's second constraint
applies — the approval design is built then, informed by how operator dispatch has actually been
used, rather than inherited from a queue kept idle in anticipation.

Two things this ADR deliberately does not settle. The manifest envelope — `r_cfg`'s fail-open
default and the unsent `ir_cfg` — is the forwarding-relay-and-manifest decision's, constrained here
only by §4.4. And whether commanded responses should carry an explicit ramp schedule rather than
relying on the agent's default ladder is an implementation question for the capability brief, not a
fork.

* [`../decision-backlog.md`](../decision-backlog.md) — decisions 5 and 7
* [`architecture-style.md`](architecture-style.md) — §1.2 on deferred capabilities, §4.2 assertions 3
  and 5
* [`../rebuild/carry-over-rules.md`](../rebuild/carry-over-rules.md) — rule 5 on background work, and
  the row-claiming precedent that leaves with the queue
* [`../investigations/2026-08-05-endpointmgr-only-retirement-audit.md`](../investigations/2026-08-05-endpointmgr-only-retirement-audit.md)
  — the coupling map, §5.4 for the disrupt surface
