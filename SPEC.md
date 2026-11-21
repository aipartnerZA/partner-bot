# SPEC — Partner Bot, Phase 1 (FileMonitor v0.1)

**Status:** planning complete, no code written. This file is both the phase plan and the progress
ledger. Design authority is [`docs/specifications/file-monitor.md`](docs/specifications/file-monitor.md);
this file never overrides it — if they disagree, the specification wins and this file is wrong.
**Phase owner:** Wes Bramwell · **Driver:** Marty (Hermes) · **Last updated:** 2026-11-21

---

## How to resume (read this first)

If you are an agent picking this up cold:

1. Read this file top to bottom — it is short by design.
2. Read [`docs/specifications/file-monitor.md`](docs/specifications/file-monitor.md) **only** for the
   task you are about to do (each task below names the sections it needs). Do not read the whole
   spec to start a task; it will eat your context budget.
3. Find the first task row in **§9 Progress** that is not `DONE` or `RETIRED`. That is your task.
4. Read its **Handoff** cell from the previous task and the last **Sweep log** line.
5. Do that one task, run its gate, record the result, tick the row, append a sweep log line, write
   the handoff note. Then stop.
6. A task marked `BLOCKED` is not yours to fix unless its blocker is resolved. Report it.

Never trust a `DONE` with no matching sweep-log line and no commit whose message starts with the task
id (`git log --oneline --grep='^T3:'`).

---

## 1. What Partner Bot is

An open-source bot that runs on a user's own Linux machine: it hosts autonomous coding agents across
several project workspaces and exposes a deliberately simple web UI (activity, files changed,
sessions). It is the only public repository in the A.I Partner fleet — everything the world sees of
our work is this repo, so its docs, licence and commit history are part of the product.

**Scope of this phase: one component only.** `partner_bot.file_monitor` — the shared filesystem
change signal that the UI, the audit log and the agent supervisor all consume. The bot shell, the
web UI and the agent runtime are later specs and are **out of scope here**.

Definition of done for phase 1: a pip-installable package whose FileMonitor watches N workspace
roots with one kernel watch per directory, fans events out to multiple subscribers with honest gap
signalling, reconciles state on a schedule, reports health and metrics, and has a test suite that
runs on a bare `python -m venv` with no network.

---

## 2. Execution rules (non-negotiable)

1. **One task per session.** Never work two tasks in one session. Never "while I'm here" edit
   another task's files.
2. **Harness:** `opencode` CLI, model `open-design/deepseek-v4.1-flash#max`, fresh session per
   task, no `--continue` across tasks. Verified working on this VPS.
3. **The 30 % context rule.** If the session's context usage passes **30 %** (the opencode status
   line shows it), stop immediately: write the handoff note for the task in §9, leave the tree
   compiling, and exit with the words `CONTEXT_LIMIT` in your final message. Do not start another
   file. A fresh session reads the handoff and continues. Two 30 % sessions are cheaper than one
   80 % session that starts hallucinating.
4. **Gates are frozen.** Gate scripts are written in T1 and must not be edited by a later task to
   make itself pass. If a gate is genuinely wrong, mark the task `BLOCKED` with the exact
   contradiction. The driver hashes each gate script when its task passes; a changed hash is a
   failed task.
5. **A gate must be shown failing first.** Before you implement, run the gate and paste the failure.
   A gate first seen green proves nothing.
6. **No agent may report a task complete on "the code looks right".** Paste real command output.
7. **Commit per task**, message `T<n>: <what changed>`, body includes the gate command and its exit
   code. Do not commit unrelated files. Do not rewrite history.
8. **No sleeping, polling or background watchers** inside a task. Tests that need elapsed time use
   `asyncio.wait_for`, never `time.sleep(30)`.
9. **No new dependency** without an ADR entry; the allowed set is `watchfiles`, `pytest`,
   `pytest-asyncio` (dev). Everything else is stdlib.
10. **No absolute host paths in logs or events.** `rel_path` only.
11. **Never invent a measurement.** If you did not run it, write `not measured`.

---

## 3. Decisions taken (bind the build)

| # | Decision | Consequence in code |
|---|---|---|
| D1 | Open source, public repo, Apache-2.0 licence unless owner overrides (Q1) | `LICENSE`, `pyproject` metadata |
| D2 | Import name `partner_bot`, module `partner_bot.file_monitor` | every import path and test |
| D3 | Backend: `watchfiles` 1.3.0, pinned `>=1.3,<2`, **no second backend** | no `polling.py`, no runtime backend switch |
| D4 | Embedded module, no daemon, no IPC, no broker | single-process tests |
| D5 | One kernel watch per directory, shared across all subscribers | watch-count assertions via `/proc/self/fdinfo` |
| D6 | Per-subscriber bounded queue; default overflow `mark_gap`; gaps are events | `GAP` kind, never silent drop |
| D7 | Never claim completeness; reconciliation is the completeness mechanism | `RESYNC_REQUIRED`, health states |
| D8 | Renames are **not** available in the live stream (ADR-002); `RENAMED` comes from the reconciler only | no rename handling in `router.py` |
| D9 | Events carry `rel_path` only; absolute paths never leave the module | normaliser strips root |
| D10 | Async delivery; user code never runs in the ingest task | consumer tasks per subscription |

Open questions Q1–Q5 live in the specification §22. **Q1 (licence) and Q2 (import name) block T1;
Q4 (rename pairing) blocks T3; Q3 and Q5 block T6.** If an answer is not in this file when a blocked
task comes up, mark it `BLOCKED` naming the question — do not choose for the owner.

---

## 4. Gate registry mechanics

```
scripts/gate.sh            driver: `bash scripts/gate.sh T3` → runs scripts/gates/T3.sh
scripts/gates/T1.sh … T8.sh
```

Rules: every gate script starts with

```bash
#!/usr/bin/env bash
set -uo pipefail
cd "$(dirname "$0")/../.."
```

and exits `0` only when its acceptance holds. T1 writes **all eight** gate scripts, each a stub that
prints `REFUSED_UNIMPLEMENTED <id>` and exits `1`, except `T1.sh` which implements T1's own checks.
`scripts/gate.sh` dispatches and propagates the exit code; unknown id → exit `2`.

Untouched by later tasks: the driver records `sha256sum scripts/gates/*.sh` when a task passes.

---

## 5. Tasks

Every task is sized for one `deepseek-v4.1-flash#max` session at ≤30 % context: at most ~10 files and
~600 lines changed, one pytest module, one gate.

### T1 — Scaffold, gate registry, CI

- **Goal:** importable empty package, pinned toolchain, gate registry, CI that runs the gates.
- **Read:** spec §18 (structure), §17 (deps), §2 (goals).
- **Files:** `pyproject.toml` (hatchling or setuptools, `requires-python >=3.11`, dep
  `watchfiles>=1.3,<2`, extras `dev = [pytest, pytest-asyncio]`), `partner_bot/__init__.py` (exports
  `__version__ = "0.1.0"` only), `partner_bot/file_monitor/__init__.py` (empty `__all__` for now),
  `README.md`, `LICENSE` (Apache-2.0 text), `.gitignore`, `scripts/gate.sh`, all eight
  `scripts/gates/*.sh` stubs, `.github/workflows/ci.yml` (uv or pip install -e ".[dev]", run
  `bash scripts/gate.sh T1`).
- **Acceptance:** `python -c "import partner_bot; print(partner_bot.__version__)"` → `0.1.0`;
  `bash scripts/gate.sh T1` → 0; `bash scripts/gate.sh T2` → 1 with `REFUSED_UNIMPLEMENTED`.
- **Gate:** `bash scripts/gate.sh T1` (checks: package imports, version string, pyproject parses,
  `pip install -e .` succeeded in the venv, all eight gate files exist and are executable, no file
  over 400 lines, grep finds no absolute `/root` or `/home` paths in source).
- **Watch out:** do not create `models.py` etc. in this task; the next task owns them.

### T2 — Event contract and configuration

- **Goal:** `models.py` (frozen slotted `FileEvent`, `EventKind`, `Health`, `MetricsSnapshot`,
  `ReconcileReport`), `config.py` (`FileMonitorConfig` + `from_env`), `exceptions.py`.
- **Read:** spec §5.1, §11, §5.4.
- **Files:** the three modules + `tests/unit/test_models.py`, `tests/unit/test_config.py`.
- **Acceptance:** `FileEvent` is immutable (`FrozenInstanceError` on assign), uses `__slots__`
  (no `__dict__`), `EventKind` has exactly the nine members of §5.1; config defaults match §11's
  table field for field (test asserts each default by name); unknown env keys do not crash.
- **Gate:** `bash scripts/gate.sh T2` → `pytest tests/unit -q` plus a `python -c` asserting
  frozen/slots/`EventKind` membership and that every default in §11 appears in the config dataclass.
- **Watch out:** no I/O in `models.py`; no import of `watchfiles` anywhere in this task.

### T3 — Watchfiles backend

- **Goal:** `backend.py` — a `Backend` protocol (`start_root`, `stop_root`, `stop_all`) and
  `WatchfilesBackend` owning one `awatch` task per root, mapping raw `(Change, path)` batches,
  `ignore_permission_denied=False`, `debounce`/`step` from config, and a `restart with backoff`
  supervisor that surfaces errors as callbacks.
- **Read:** spec §3.3, §5.2, §5.3, §9.1, ADR-002.
- **Files:** `backend.py`, `tests/integration/test_backend.py`.
- **Acceptance:** start on a `tmp_path` tree → write a file → a mapped batch arrives within 2 s;
  `stop_all()` leaves **zero** `inotify wd:` lines in `/proc/self/fdinfo`; a directory created after
  start is watched (write inside it is seen within 2 s); a forced exception marks the root
  unavailable and is retried.
- **Gate:** `bash scripts/gate.sh T3` → runs `tests/integration/test_backend.py` and asserts the
  fdinfo watch count returns to 0 after teardown.
- **Watch out:** Q4 — if the owner says rename pairing must be live, stop; ADR-002 has to change
  first. Do not add a watchdog backend on your own initiative.

### T4 — Manager, workspaces, subscriptions

- **Goal:** `manager.py` — `FileMonitor` façade (`start`, `stop`, `register_workspace`,
  `unregister_workspace`, `subscribe`, `metrics`, `health`), root minimalisation (nested roots
  collapse), realpath containment checks, per-subscription queue creation.
- **Read:** spec §7.1, §10, §6, D5, D9.
- **Files:** `manager.py`, `tests/unit/test_workspaces.py`, `tests/integration/test_shared_watches.py`.
- **Acceptance:** `/srv/a` + `/srv/a/b` → one backend root; two subscriptions on one workspace → one
  root and `watched_directories == dir count`; `subscribe` with an unregistered id raises
  `WorkspaceError`; `..` and absolute paths are rejected; `stop()` twice is safe and leaves zero
  watches.
- **Gate:** `bash scripts/gate.sh T4` → those two test modules plus the fdinfo assertion.

### T5 — Router: queues, coalescing, gaps

- **Goal:** `router.py` — `matches()`, `put_nowait` dispatch, the three overflow policies, the
  coalescing truth table of §7.3, `GAP` emission with missed count, per-subscription ordering, and
  consumer tasks that isolate subscriber exceptions.
- **Read:** spec §7.2–7.4, §8.2, §8.4, ADR-005, ADR-006.
- **Files:** `router.py`, `tests/unit/test_coalescing.py`, `tests/concurrency/test_slow_subscriber.py`.
- **Acceptance:** the six-row coalescing table reproduces exactly; a 10 000-event burst into a
  100-slot queue yields `GAP` with `missed > 0` and RSS growth bounded (assert `< 5 MB` delta); one
  slow consumer does not delay a fast one by more than 100 ms (asserted with timestamps); a
  subscriber raising in its handler is disconnected and counted, others unaffected.
- **Gate:** `bash scripts/gate.sh T5`.
- **Watch out:** ordering test must not rely on cross-path kernel order (§7.4).

### T6 — Reconciliation

- **Goal:** `reconciliation.py` — sqlite manifest (WAL, one file per workspace under `state_dir`),
  streaming scan, diff, synthetic events, inode-based `RENAMED` inference, startup/interval/on-demand
  triggers with the entry thresholds of §9.2, cooperative yielding.
- **Read:** spec §9.2, §5.4, ADR-008.
- **Files:** `reconciliation.py`, `tests/integration/test_reconcile.py`.
- **Acceptance:** first run creates the manifest and reports `created == entry count`; deleting a file
  then reconciling reports `deleted == 1` and emits one `DELETED`; `os.rename` then reconciling emits
  one `RENAMED` with `src_rel_path`; a 5 000-file scan yields at least once per 2 000 entries
  (instrumented counter, not wall-clock); a missing root reports a failure without raising out of
  `reconcile()`.
- **Gate:** `bash scripts/gate.sh T6`.
- **Watch out:** Q3 (state dir) and Q5 (hashing) block this task. Do not hash contents.

### T7 — Metrics, health, logging

- **Goal:** `metrics.py` + health wiring: exact `watched_directories` from fdinfo, latency ring
  buffer, all §12 counters, the four health states with the conditions from the §12 table, JSON log
  formatter, `log_events` sampling.
- **Read:** spec §12, §9.1.
- **Files:** `metrics.py`, `partner_bot/file_monitor/logging.py` (or `log.py`), `tests/unit/test_health.py`.
- **Acceptance:** health is `HEALTHY` on a clean start; `DEGRADED` after a forced `GAP`;
  `RECONCILE_REQUIRED` after `RESYNC_REQUIRED`; `UNAVAILABLE` when a root's task is cancelled; each
  transition asserted; log records parse as JSON and contain no absolute path (asserted).
- **Gate:** `bash scripts/gate.sh T7`.

### T8 — Docs, examples, 0.1.0 release prep

- **Goal:** README that a stranger can follow (what it is, install, 10-line quickstart, guarantees
  ladder from §9.3 verbatim, supported/unsupported list from §2.2 and §21), `docs/api.md`,
  the seven `examples/` from §6.1 as runnable scripts, `CHANGELOG.md` with `0.1.0`, version bump
  check, and a status line flip in `docs/specifications/file-monitor.md` from "nothing implemented"
  to "implemented in v0.1.0, see CHANGELOG".
- **Read:** spec §6.1, §9.3, §21, §23.
- **Files:** `README.md`, `docs/api.md`, `examples/*.py` (7), `CHANGELOG.md`, `docs/specifications/file-monitor.md` (status line only).
- **Acceptance:** full `pytest -q` green; every example runs headless under
  `timeout 20 python examples/NN_*.py` and exits 0; README's quickstart block is copy-paste runnable
  in a fresh venv (gate executes it against `tmp_path`); `CHANGELOG` names 0.1.0 with the four
  guarantees actually shipped.
- **Gate:** `bash scripts/gate.sh T8`.
- **Watch out:** examples must not require network or a real project directory.

**Not in this phase:** dashboard, SSE/WebSocket transport, Git diff module, agent attribution,
benchmark suite (`bench/`, phase 3), packaging wheels for release (phase 5).

---

## 6. Verification the driver performs (not the worker)

After each task the driver — not the agent — does all of:

1. `git log --oneline -1 --grep="^T<n>:"` — a commit exists for this task.
2. `sha256sum scripts/gates/*.sh` compared to the recorded hashes — no gate was edited.
3. `bash scripts/gate.sh T<n>` re-run from a clean shell; exit code recorded.
4. `git diff --stat HEAD~1` inspected: files touched must be inside the task's declared file set.
5. `pytest -q` full suite re-run for regressions.
6. Ticks the §9 row and appends the sweep log line itself if the worker forgot.

A worker's `DONE` is a claim; the six checks above are the evidence.

---

## 7. Fault tasks (pre-declared, expected)

| Fault | First task that will hit it | Instruction |
|---|---|---|
| `watchfiles` wheel unavailable for the interpreter | T1/T3 | build from source needs Rust; if so, mark BLOCKED and report — do not switch backend |
| `awatch` raises `ENOSPC` | T3/T4 | this is the watch-limit path; assert `UNAVAILABLE`, do not silently shrink the tree |
| fdinfo parsing differs per kernel | T4/T7 | parse `inotify wd:` lines, never a fixed offset; if absent, skip with `pytest.skip` and record |
| sqlite locked under concurrent reconcile | T6 | WAL + one reconcile at a time; assert the second `reconcile()` awaits, not errors |
| test flakiness from 40 ms latency | T3–T5 | use `asyncio.wait_for(..., timeout=2.0)`; never a bare assert on arrival order across paths |

---

## 8. Roadmap beyond this phase (context only — do not start)

- **P2 Shared resource management** — already largely in P1; remainder: overlapping-workspace routing
  edge cases, per-workspace metrics.
- **P3 Reliability hardening** — overflow-injection harness, restart storms, 24 h soak, benchmark
  suite (`bench/`), tuning constants from measurements.
- **P4 Platform integration** — agent supervisor subscriptions, dashboard SSE, audit consumer,
  Git-backed diff module.
- **P5 Release** — wheels, `pip install partner-bot`, docs site, `v0.1.0` tag, announcement.
- **P6 Bot shell + simple UI** — separate spec (`docs/specifications/bot-shell.md`, not written).

---

## 9. Progress

Statuses: `TODO`, `DONE`, `BLOCKED`, `RETIRED`.

| # | Task | Status | Gate | Evidence | Handoff (what the next session must know) |
|---|---|---|---|---|---|
| T1 | Scaffold, gate registry, CI | TODO | `bash scripts/gate.sh T1` | — | — |
| T2 | Event contract and configuration | TODO | `bash scripts/gate.sh T2` | — | — |
| T3 | Watchfiles backend | TODO | `bash scripts/gate.sh T3` | — | — |
| T4 | Manager, workspaces, subscriptions | TODO | `bash scripts/gate.sh T4` | — | — |
| T5 | Router: queues, coalescing, gaps | TODO | `bash scripts/gate.sh T5` | — | — |
| T6 | Reconciliation | TODO | `bash scripts/gate.sh T6` | — | — |
| T7 | Metrics, health, logging | TODO | `bash scripts/gate.sh T7` | — | — |
| T8 | Docs, examples, 0.1.0 prep | TODO | `bash scripts/gate.sh T8` | — | — |

### Handoff note template (paste into the row's Handoff cell)

```
DONE|BLOCKED <task> <YYYY-MM-DD HH:MM UTC> — files: <list> — gate: <command> exit <n> (<one line of output>)
next: T<n+1> — must know: <the one thing that would cost the next session an hour>
open: <unresolved issue, or "none"> — context used: <xx>%
```

---

## 10. Sweep log

Newest last. One line per session.

```
2026-11-21 planning  — spec written (this file + docs/specifications/file-monitor.md); no code; next: T1
```
