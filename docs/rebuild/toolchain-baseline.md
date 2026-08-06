# Toolchain baseline

The work implied by [`../adr/language-and-framework-baseline.md`](../adr/language-and-framework-baseline.md). Unlike [`backend-carry-over-rules.md`](backend-carry-over-rules.md), which governs code as it is carried, most of this runs **before** any code is carried: it establishes the interpreter, the lock file and the boundary tooling that the carry-over rules then check against.

The ADR decides the baseline. This document is how it is stood up, and what to look at when a step fails.

## 1. Interpreter coherence

Three artifacts state the interpreter and all three must agree. Today none of them does: `requirements.txt` was compiled with 3.12, the images run 3.11, and nothing declares a requirement at all.

| Artifact | Value |
|---|---|
| `pyproject.toml` | `requires-python = ">=3.14"` |
| Backend images | `python:3.14-slim-bookworm` |
| Lock file | recompiled **on 3.14** |

The lock recompile is the rebuild's only version-change event, and it must not become an upgrade. pip-compile preserves the pins already in its output file unless `--upgrade` is passed, so run it without that flag: versions hold and only the marker resolution changes. The two commands are otherwise identical and produce very different diffs, so check the result — a recompile that moved library versions was an upgrade, and belongs to the later phase rather than to this one.

The current lock carries **zero** `python_version` markers, which is the artifact of resolving them away for one interpreter. Expect the recompiled file to carry markers wherever the dependency set declares them; their appearance is the fix landing, not a regression.

## 2. Dependency readiness

**Checked 2026-08-06.** This snapshot dates faster than anything else here — re-derive it rather than trust it if the pins have moved.

Every pinned dependency that ships compiled artifacts is already at or past its first release with Python 3.14 support:

| Pin | Status on 3.14 |
|---|---|
| `asyncpg==0.31.0` | first release with cp314 wheels |
| `psycopg2-binary==2.9.12` | first release with cp314 wheels |
| `greenlet==3.5.1` | cp314 and cp315 wheels, including free-threaded |
| `cryptography==50.0.0` | covered by `cp311-abi3`, forward-compatible by construction |
| `sqlalchemy==2.0.51` | 3.14 support from 2.0.41; ORM annotation scanning adjusted in 2.0.48 |
| `pydantic==2.13.4` | 3.14 support from 2.12 |

`greenlet` is the one worth re-checking first on any future interpreter move: it manipulates the C stack, so it needs per-version work and has historically been the last of this set to publish for a new CPython. SQLAlchemy's async mode depends on it, so it gates the whole backend rather than one module.

Free-threaded wheels appear in several of these sets and are incidental. Backend concurrency is asyncio and I/O-bound; free-threading is not a factor in any decision here.

## 3. The verification pass

Recompile, install, and run the suite. The result is expected to be green — §2 is why — so treat this as a checkbox with one specific thing to watch.

3.14 defers annotation evaluation (PEP 649/749), and the two libraries that inspect annotations most heavily are SQLAlchemy's `Mapped[]` declarative and Pydantic's model construction. So importing the application is not sufficient evidence: run the tests that build ORM models and validate request bodies. A failure surfaces there and nowhere else.

If a dependency genuinely cannot resolve on 3.14, the fallback is 3.13 and it changes one line in each of §1's three artifacts. [`../adr/language-and-framework-baseline.md`](../adr/language-and-framework-baseline.md) §4.1 covers why that does not reopen the decision.

## 4. HTTP client surface

The backend uses **`httpx`**, which is the only one of the three clients in the current dependency set spanning both sync and async — `requests` is sync-only, `aiohttp` async-only — so it absorbs both without a second client. It is already the majority (5 files in `endpointmgr`, 1 in `shared`) and is the test client regardless.

Both other clients leave with the descope rather than by refactor, bar one file:

| Usage | Disposition |
|---|---|
| `aiohttp` — 7 files in `analysis` | Out of scope |
| `aiohttp` — `endpointmgr/services/alert_forwarder.py` | Out of scope; posts to the Analysis service's `/logs` (`:29-32`) and is the forwarding task named in [`backend-carry-over-rules.md`](backend-carry-over-rules.md) rule 5 |
| `aiohttp` — `endpointmgr/services/intunewin_service.py` | **Convert to `httpx`** |
| `requests` — `shared/graph_notifier.py` | Out of scope; [`backend-carry-over-rules.md`](backend-carry-over-rules.md) rule 7 routes it nowhere |

So one conversion, and the dependency set loses both `aiohttp` and `requests` as a consequence. The surviving usage is a plain `ClientSession()` post with a timeout, with no WebSocket use — the one gap that would have made `httpx` insufficient.

## 5. Boundary tooling to stand up

None of this exists today. `.pre-commit-config.yaml` carries only secret-scanning and file hygiene, there is no lint configuration, and `pyrightconfig.json` points a single `extraPaths` root at the flat namespace.

- **import-linter**, configured with the contracts of [`../adr/architecture-style.md`](../adr/architecture-style.md) §4.2 assertions 1 and 2. The ADR names the tool; the contracts are build work, and rule 2 of [`backend-carry-over-rules.md`](backend-carry-over-rules.md) is the rule currently resting on a check that does not exist.
- **`pyproject.toml` module roots**, which §4.2 assertion 8 needs in order to have anything to declare a file outside of. Writing them retires the four `sys.path.insert` calls.
- **The lock/image interpreter check** of [`../adr/language-and-framework-baseline.md`](../adr/language-and-framework-baseline.md) §4.3 assertion 2. Cheap, and it is the check whose absence produced §1.

## 6. Related

- [`../adr/language-and-framework-baseline.md`](../adr/language-and-framework-baseline.md) — the decision this implements, including the version policy and what is out of scope
- [`../adr/architecture-style.md`](../adr/architecture-style.md) — §4.2, the assertions the tooling in §5 exists to run
- [`backend-carry-over-rules.md`](backend-carry-over-rules.md) — what happens to code once this is in place
