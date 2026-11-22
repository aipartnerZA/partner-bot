# Partner Bot

An open-source bot that runs on your own Linux machine. It hosts autonomous coding agents across
several project workspaces and gives you one deliberately simple web UI: what ran, what changed,
what needs you.

> **Status: design stage.** Nothing is runnable yet. Phase 1 (the FileMonitor core) is fully
> specified and is being built task by task. See [`BUILD-PLAN.md`](BUILD-PLAN.md) for the live plan and
> [`docs/specifications/file-monitor.md`](docs/specifications/file-monitor.md) for the architecture.

## Why

Running coding agents on a VPS is cheap. Knowing what they actually did to your files is not.
Partner Bot's first component, **FileMonitor**, is a shared filesystem-change signal: one kernel
watch set per directory no matter how many agents or panels subscribe, bounded memory, honest gap
reporting, and periodic reconciliation so "nothing changed" is a claim backed by evidence rather than
by an empty event queue.

It deliberately does **not** pretend to know things the kernel cannot tell it. Renames arrive as
delete+create. Queue overflow is not observable through the Python API it builds on. Those limits are
written down in the specification and are the reason reconciliation exists.

## What phase 1 delivers

- `partner_bot.file_monitor` — embeddable async component, no daemon, no broker.
- One `inotify` watch per directory, shared by every subscriber (measured: 241 directories → 241
  watches, one instance).
- Immutable events; per-subscriber bounded queues; explicit `GAP` events instead of silent loss.
- SQLite reconciliation manifest with inode-based rename inference.
- Four health states and measurable metrics, no estimates.

Not in phase 1: the web UI, the agent runtime, content diffing, and any claim of exactly-once
delivery.

## Requirements

- Linux (Ubuntu tested), Python 3.11+
- `inotify` watch headroom: check `sysctl fs.inotify.max_user_watches`. A stock Ubuntu box allows
  65 535 watches, which is roughly 65 535 directories.

## Install (once phase 1 lands)

```bash
python -m venv .venv && . .venv/bin/activate
pip install -e ".[dev]"
pytest -q
```

## Layout

```
docs/specifications/file-monitor.md   architecture, researched and measured
BUILD-PLAN.md                         phase-1 plan, gates, progress ledger
partner_bot/file_monitor/             the component (phase 1)
tests/                                unit, integration, concurrency, failure
examples/                             runnable API examples
```

## Licence

Apache-2.0. See [`LICENSE`](LICENSE).
