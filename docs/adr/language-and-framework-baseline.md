---
status: "accepted"
date: 2026-08-06
scope: all backend modules, the CI enforcement surface, build and runtime images
related: architecture-style, ADR-M3-001, ADR-M3-004
supersedes:
---

# Python 3.14 with FastAPI and SQLAlchemy as the backend baseline

## 1. Context and Problem Statement

### 1.1 The baseline is assumed everywhere and stated nowhere

The backend's language, framework and ORM are load-bearing for decisions already taken, and no artifact declares them. [`architecture-style.md`](architecture-style.md) §4.2 specifies an enforcement surface that only exists in Python tooling; rule 2 of [`../rebuild/backend-carry-over-rules.md`](../rebuild/backend-carry-over-rules.md) names a check that does not exist yet.

Meanwhile nothing pins an interpreter. There is no `pyproject.toml` and no `requires-python` anywhere, the five packages are siblings in one flat namespace assembled by four `sys.path.insert` calls (`EagerBeaver/src/{analysis,responder,endpointmgr,forwardingrelay}/main.py`), and the dependency lock was resolved against an interpreter nothing runs — `requirements.txt` is compiled with Python 3.12 while every image is `python:3.11-slim-bookworm` (`EagerBeaver/src/requirements.txt:2`, `EagerBeaver/Dockerfile.endpointmgr:2`).

That lock carries zero `python_version` markers, because pip-compile resolves markers away for the interpreter it runs on. The file is therefore valid for exactly one interpreter and installed on another, `--require-hashes` succeeds regardless, and no artifact exists that would fail on the mismatch.

### 1.2 The backend is built by restructuring an existing body of Python

The implementation is a relocation of code that already exists, under invariants that already have tests specified. Connection acquisition moves to one component across 42 of 72 database-touching files; 81 inlined `select()` calls move into repositories; 48 table declarations move to owning modules; a 90-file, 35,047-line shared layer splits three ways ([`../rebuild/backend-carry-over-rules.md`](../rebuild/backend-carry-over-rules.md) rules 1, 2, 4, 7).

Each of those is a mechanical move with a check behind it. Under a different language every one becomes a reimplementation from a reference, and the largest single unit — `endpointmgr/api/agent_registration.py` at 2,656 lines — stops being a refactor and becomes reverse engineering.

### 1.3 The behaviour being preserved cannot be renegotiated

Deployed agents already speak ~25 `/api/v1/endpointmgr/…` paths. The wire contract is a standing constraint, so exact behavioural preservation is a requirement rather than a goal, and carrying the implementation is the cheapest way to obtain it.

### 1.4 The enforcement surface is language-specific

[`architecture-style.md`](architecture-style.md) §4.2 assertions 1 and 2 are import-linter contracts; assertion 3 walks SQLAlchemy metadata; assertion 7 asserts that `select(`, `update(` and `delete(` appear only in repository packages, which is a check on SQLAlchemy 2.x constructs and matches nothing under the 1.x `session.query()` idiom. Changing language or ORM major version does not adjust that decision's confirmation — it removes it.

### 1.5 The deferred capabilities are ecosystem-bound

Alert enrichment and automated response are deferred, not abandoned ([`architecture-style.md`](architecture-style.md) §1.2), and both are built on LangChain and LangGraph. `ADR-M3-001` already weighed a rewrite against this and rejected it, naming loss of the Python AI/ML ecosystem alongside the effort. Its finding stands and is not re-derived here; §6.1 records it with the rest of the estate's prior art.

## 2. Decision Drivers

* The implementation is a restructure, so verification effort concentrates on behaviour that moved. A second, simultaneous source of behavioural change makes failures unattributable.
* The agent wire contract fixes the observable surface; nothing about it is negotiable.
* An accepted enforcement surface exists and assumes Python tooling.
* Deferred capabilities return into whatever ecosystem the baseline chooses.
* A baseline that pins more than it needs to becomes wrong on its next patch release.

## 3. Considered Options

* **A — Python baseline, majors pinned.** Declare the language, framework, ORM and boundary tooling; leave everything below a major version to the lock.
* **B — Rewrite the backend in Go or Rust.**
* **C — Polyglot: new modules in another language behind the wire contract.**

Source-protection measures — Cython compilation, obfuscation — are not options here. They change how the artifact is distributed rather than what it is written in, and they were evaluated against the deployment model instead. §6.1 records them and their outcome.

## 4. Decision Outcome

Chosen option: **A**, because it is the only option under which the specified enforcement surface remains valid and the restructure stays a restructure. B and C both convert a scoped relocation into an open-ended reimplementation of behaviour that cannot be renegotiated, and C additionally forces module boundaries to become language boundaries, arriving at option D of [`architecture-style.md`](architecture-style.md) §3 by a route that decision rejected on its merits.

The baseline is:

| Element | Constraint | Why it is pinned here |
|---|---|---|
| Interpreter | Python 3.14 | §4.1 |
| Web framework | FastAPI with Uvicorn | Serves the wire contract |
| ORM | SQLAlchemy 2.x, async, with asyncpg | §1.4 — assertions 3 and 7 are specific to it |
| Validation | Pydantic 2.x | The wire boundary's schema; the v1→v2 break crosses it |
| Test framework | pytest with pytest-asyncio | §4.2 assertions 3–7 are tests |
| Boundary tooling | import-linter | §4.2 assertions 1 and 2 *are* import-linter contracts |
| Project metadata | `pyproject.toml` declaring `requires-python` and module roots | §4.2 assertion 8 fails on a file outside a declared module root, which requires roots to be declared |

Nothing below a major version is decided here. Minor and patch versions are the lock file's, and the lock file is the authority on them.

### 4.1 The interpreter, and the rule that selected it

Python 3.14 (released 2025-10-07), because it is the newest release in upstream bugfix support that the dependency set resolves on. It holds bugfix support into 2027 and reaches end-of-life in October 2030.

The rule is the decision; 3.14 is its answer as of this date. Applying it excludes 3.12 and 3.11, both security-only, reaching end-of-life in October 2028 and October 2027 — either would ship with roughly two years of runway and pay its migration across the whole codebase at a time not of our choosing. It excludes 3.13, still in bugfix but returning to security-only around the 3.15 release in October 2026. It excludes 3.15 itself, which arrives too near the build to be a known quantity.

If the dependency set fails to resolve on 3.14, the same rule selects 3.13 and this decision does not reopen — the criterion is what was decided, not the number.

### 4.2 Consequences

**Good**

* The restructure's failures are attributable, because library behaviour is held still while code moves.
* `requires-python` makes the §1.1 mismatch a class of defect that fails loudly rather than silently.
* The deferred capabilities re-enter an ecosystem that already carries them.

**Neutral**

* The dependency set is already at versions that support 3.14, so adopting it changes the interpreter without moving any library. Evidence and its date are in [`../rebuild/toolchain-baseline.md`](../rebuild/toolchain-baseline.md) §2.
* Declaring `pyproject.toml` retires the flat-namespace `sys.path.insert` assembly of §1.1 as a side effect of needing module roots.

**Bad**

* Backend source remains readable wherever a customer can reach the filesystem. This is a real cost under the two-deployment-shapes constraint, since self-hosted is first-class rather than a degraded mode. Containment at the infrastructure layer is the accepted answer and this decision adds nothing to it; the source-level mitigations were evaluated and rejected on their own terms (§6.1). One of those rejections runs partly on this baseline — Cython was declined in part for FastAPI and Pydantic compatibility risk — so choosing FastAPI and Pydantic is also choosing not to have that mitigation available.
* Ecosystem concentration deepens. Every deferred capability returning on LangChain or LangGraph makes a future language change more expensive than it is today.

### 4.3 Confirmation

1. **The declared interpreter is the installed interpreter.** `pyproject.toml` declares `requires-python`, and installation fails on an interpreter that does not satisfy it. Enforced by the packaging tool rather than by a CI job.
2. **The lock file was compiled on the interpreter the images run.** A CI check comparing the `# autogenerated by pip-compile with Python 3.X` header in `requirements.txt` against the `python:3.X` tag in the images. This is the check whose absence produced §1.1; it fails on a recurrence.
3. **Boundary contracts run in CI.** [`architecture-style.md`](architecture-style.md) §4.2 assertions 1 and 2, which is where they are specified and tracked.

The amendment obligation in §4.4 has no check behind it. It is a review obligation, and stating it that way is deliberate — no test can distinguish a deliberate major bump from an inadvertent one.

### 4.4 Constraints Imposed

* Minor and patch versions are not decided here and need no amendment to change.
* A **major** version bump that would change a [`architecture-style.md`](architecture-style.md) §4.2 assertion returns here as an amendment. Assertion 7 is the live example: an ORM major version that alters query construction invalidates the check, not merely the code it inspects.
* Library version bumps are **out of scope for the rebuild** and belong to a later phase. The one version-change event the rebuild performs is recompiling the lock on the new interpreter, which is a marker-resolution fix and not an upgrade — [`../rebuild/toolchain-baseline.md`](../rebuild/toolchain-baseline.md) §1 states how it is kept to that.
* The AI/ML libraries are deliberately absent from §4. They arrive with the module that needs them and are that module's choice; anything pinned for them now would be stale before it is used.

## 5. Pros and Cons of the Options

### 5.1 B — Rewrite the backend in Go or Rust

**Pros**

* Compiled artifacts answer the §4.2 source-exposure consequence directly rather than by containment.
* A single-language estate with the agent shares tooling, CI and expertise.
* Lower runtime footprint in the appliance.

**Cons**

* Duplicates work already scoped: every carry-over rule becomes a reimplementation against a frozen wire contract, verified without the original as a running reference.
* Removes the confirmation surface of an accepted decision (§1.4), reopening it.
* Loses the ecosystem the deferred capabilities are built on. Both `ADR-M3-001` and `ADR-M3-004` reached this conclusion previously, on effort and on not maintaining a second API surface respectively (§6.1).

### 5.2 C — Polyglot: new modules in another language behind the wire contract

**Pros**

* No big-bang. The wire contract is a natural seam, and performance-sensitive modules could move individually.

**Cons**

* Module boundaries become process and language boundaries, which is option D of [`architecture-style.md`](architecture-style.md) §3 reached indirectly.
* §4.2's assertions cannot span two toolchains, so boundary enforcement degrades to the subset each language can express.
* Two build, test and dependency surfaces for a team of this size.

## 6. More Information

### 6.1 Options previously considered elsewhere

The estate has prior art on this decision, all of it reached while answering a different question — how to protect intellectual property in customer-controlled deployments. It is recorded here so the reasoning is not re-derived, and so a reader can see which rejections still hold.

| Option | Rejected because | Recorded in |
|---|---|---|
| Go/Rust rewrite | 6+ months of effort; loses the Python AI/ML ecosystem (LangChain); duplicates the agent's Rust investment | `ADR-M3-001` |
| Cython compilation | Build complexity, FastAPI/Pydantic compatibility risk, and it only raises the bar — a determined reverse engineer still succeeds. Redundant once the VM is locked. | `ADR-M3-001`, reaffirmed in `AgileFramework/docs/epics.md:217` and `:5024`, and listed out of scope in `AgileFramework/docs/architecture-m3-llm-playbooks.md:1235` |
| PyArmor obfuscation | Easily defeated; provides a false sense of security | `ADR-M3-001` |
| SaaS-only distribution | Excludes air-gapped customers (government, defence); many enterprises require on-premise | `ADR-M3-001` |
| Legal protection only | Insufficient deterrent against well-funded competitors | `ADR-M3-001` |
| A parallel API in a second language | All user-facing features stay in one stack rather than maintaining a second API surface alongside it | `ADR-M3-004` |
| Test framework other than pytest | Leverages the existing Python test infrastructure and team skills | `AgileFramework/docs/archive/m1/architecture.md:391` |

`ADR-M3-001` and `ADR-M3-004` are subsections of `AgileFramework/docs/epics.md`; the ADR-estate decision has not yet settled how those foreign series are referenced.

Two things the sweep did not find, both worth knowing:

* **No prior art on the framework or the ORM.** Nothing in the estate compares FastAPI, or SQLAlchemy, against an alternative — the choices appear only as inventory. This ADR is the first place either is argued.
* **The one rationale on record is a single line.** `AgileFramework/docs/project-docs/technology-stack.md` opens by attributing Python to "rapid development, AI integration, and ecosystem access", and its ~360 lines of version tables carry no reasoning. Those tables have also drifted from the code they describe — they list `python-jose`, `passlib` and `structlog`, none of which appear in the dependency set, which is why §4 constrains majors and leaves versions to the lock.

### 6.2 Sources for the interpreter and dependency claims

Interpreter support states and dates are from the [Python devguide version table](https://devguide.python.org/versions/).

The claim in §4.2 that the dependency set already supports 3.14 rests on: SQLAlchemy beginning 3.14 support in [2.0.41](https://www.sqlalchemy.org/blog/2025/05/14/sqlalchemy-2.0.41-released/) and adjusting the ORM's annotation scanning for it in [2.0.48](https://www.sqlalchemy.org/changelog/CHANGES_2_0_48); Pydantic adding 3.14 support in [2.12](https://pydantic.dev/articles/pydantic-v2-12-release); and cp314 wheels published for [asyncpg](https://pypi.org/project/asyncpg/), [psycopg2-binary](https://pypi.org/project/psycopg2-binary/), greenlet and cryptography. The pinned versions clear all four thresholds. Per-package detail and the date it was checked are in [`../rebuild/toolchain-baseline.md`](../rebuild/toolchain-baseline.md) §2.

3.14 defers annotation evaluation ([PEP 649](https://peps.python.org/pep-0649/), [PEP 749](https://peps.python.org/pep-0749/)), which is why SQLAlchemy's `Mapped[]` declarative and Pydantic model construction are the surfaces the §2 verification exercises. Pydantic **v1** does not work on 3.14 or later, so a carried v1 compatibility shim would be a blocker; none is present.

### 6.3 Status

**Why this is accepted rather than proposed.** The other decisions carried in this repo hold at Proposed pending implementation evidence. This one has no empirical content to gate: the language, framework and ORM are not propositions a vertical slice can falsify. The interpreter number has a cheap check, and §4.1 makes its failure select a different number under the same criterion rather than reopen the decision.

### 6.4 Related

- [`architecture-style.md`](architecture-style.md) — the enforcement surface this baseline serves, and the module structure `pyproject.toml` declares roots for
- [`../rebuild/toolchain-baseline.md`](../rebuild/toolchain-baseline.md) — the work this decision implies, and the dated readiness evidence
- [`../rebuild/backend-carry-over-rules.md`](../rebuild/backend-carry-over-rules.md) — the restructure of §1.2, rule by rule
- `ADR-M3-001` — the deployment model whose containment answers the §4.2 source-exposure consequence, and the origin of most of §6.1
- `ADR-M3-004` — the prior decision to keep user-facing features in one stack
