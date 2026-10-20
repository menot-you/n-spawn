# CLAUDE.md — nSPAWN

Repository contract and contributor guide for `nSPAWN`, the multi-provider supervisor and SQLite run registry for native AI coding agents.
Read this file before changing code, process custody, adapters, or documentation.

## What this repository builds

`nSPAWN` dispatches, isolates, and tracks Claude Code, Codex `app-server`, and AGY sessions on authenticated host accounts with quota-aware routing, worktree locks, and credential-redacted transcripts.

| Part | Path | Responsibility |
| --- | --- | --- |
| Supervisor and admission | [supervisor.py](nspawn/supervisor.py), [mcp.py](nspawn/mcp.py), [cli.py](nspawn/cli.py) | Admission locks, idempotency keys, run lifecycle (`L1`–`L8`), MCP tools, and the `nspawn <noun> <verb>` CLI |
| Roster and quota picker | [owner_roster.py](nspawn/owner_roster.py), [picker.py](nspawn/picker.py), [quota_sources.py](nspawn/quota_sources.py) | Account discovery from `~/.nott/spawn.toml`, 5-hour and 7-day utilization windows, and `cooling_until` walls |
| Process custody and cages | [driver.py](nspawn/driver.py), [child.py](nspawn/child.py), [native.py](nspawn/native.py) | Detached driver processes, systemd slices (`launchd` on macOS), `ProcessIdentity` checks, and read/write cages |
| System1 routing and judgment | [system1.py](nspawn/system1.py), [system1 lib](packages/system1/nott_system1/__init__.py) | Task-class estimation (`EXECUTION_CLASS`) and probabilistic tools (`noul`, `choice`, `score`) |
| Persistence and IPC | [registry.py](nspawn/registry.py), [redact.py](nspawn/redact.py), [database](packages/database/nott_database/__init__.py), [socket](packages/socket/nott_socket/__init__.py) | SQLite run registry (`nspawn.sqlite3`), stream-json credential redaction, and peer-credential Unix sockets |

## Local checks and commands

Run these commands from the repository root with the pinned Python version and `uv.lock`:

| Need | Command |
| --- | --- |
| Sync workspace dependencies | `uv sync --group dev` |
| Lint and format check | `uv run ruff check .` and `uv run ruff format --check .` |
| Strict type check | `uv run mypy --strict nspawn` |
| Unit and contract suite | `uv run --group dev pytest tests/test_owner_roster.py tests/test_supervisor_layout.py -q` |
| Prepare disposable Node SDK prefix | `uv run --no-sync python tests/sdk_setup.py "$PWD/.nott/disposable/test-sdk"` |
| Wire projection check | `uv run --group dev pytest tests/test_run_effort_public.py -q` |
| Lockfile consistency | `uv lock --check` |

Every test creates its own temporary SQLite registry and synthetic account home.
Never point a test at `~/.nott/state/database` or mutate a live vendor account home.

## Architecture and code boundaries

1. **Eight-layer direction (`L1`–`L8`):** keep layer dependencies strictly acyclic as defined in the [rulebook](RULEBOOK.md). Lower layers never import higher layers.
2. **Native vendor adapters, never a proxy:** each route executes the vendor's native binary (`claude`, `codex`, `agy`) against an existing authenticated account home declared in `~/.nott/spawn.toml`.
3. **Caged readers vs. one-writer custody:** `read_only: true` runs inside a strict read-only cage confined to `cwd` and explicit `read_paths`. `native_write: true` requires a standalone Git clone, acquires an exclusive one-writer lock, isolates `HOME` and `TMPDIR` under `.nott/disposable`, and disables network access in the writer cage.
4. **Checked `ProcessIdentity` before signals:** never send `SIGSTOP` or `SIGKILL` to a bare PID. Always check `(pid, starttime_ticks, uid)` first and drain the run's systemd slice before releasing its worktree claim.
5. **Honest quota, effort, and terminal states:** report quota state as `observed`, `stale`, `expired`, `never_observed`, or `no_source` (never `0%` for unknown usage). Record `requested`, `sent`, and `executed` effort separately. Require vendor result frames before marking a run succeeded. Exit code `0` alone is insufficient.
6. **System1 task-class ceiling:** when the roster defines `[classes]`, classify the first 4,000 prompt characters through [system1.py](nspawn/system1.py). Downgrade to the observed class only when it is cheaper on every axis.
7. **Redacted persistence:** pass all vendor stream-json frames and stderr lines through [redact.py](nspawn/redact.py) before writing to `nspawn.sqlite3` via `nott_database.connect`.
8. **Grammar canon:** CLI commands and MCP tools use singular `<noun> <verb>` names (`nspawn run start`, `nspawn_run_start`) with no aliases.

## Working rules for agents and contributors

- Write all code comments, docstrings, commit messages, pull requests, and documentation in Simplified Technical English (ASD-STE100). Keep sentences under 25 words and use active voice.
- Write the regression test first for any behavior change. Observe the test fail (`RED`) before changing product code.
- Check wire recordings (`*.public.ndjson`) with [test_run_effort_public.py](tests/test_run_effort_public.py) so no private path, git remote, or token enters the repository.
- Read the [rulebook](RULEBOOK.md) before changing admission, process custody, or schema migrations.

## Canonical documentation map

| Need | Canonical document |
| --- | --- |
| Product overview and quickstart | [README](README.md) |
| Layer direction, custody, and admission invariants | [Rulebook](RULEBOOK.md) |
| Documentation index and first contribution | [Docs index](docs/README.md) |
| Service and trust boundaries | [Architecture](docs/ARCHITECTURE.md) |
| Setup, operation, backup, and restoration | [Runbook](docs/RUNBOOK.md) |
| Complete MCP tool and CLI command reference | [Reference](docs/REFERENCE.md) |
| Developer toolkit and Context7/RTK setup | [Setup](docs/SETUP.md) and [setup-catalog.json](docs/setup-catalog.json) |
| Quality gates and security review | [Quality](docs/QUALITY.md) |
| Source installation and prerequisites | [Installation](docs/INSTALLATION.md) |
| Data boundaries and external processing choices | [Privacy](docs/PRIVACY.md) |
| Schema compatibility and upgrades | [Upgrade](docs/UPGRADE.md) |
| Failure diagnosis | [Troubleshooting](docs/TROUBLESHOOTING.md) |
| Working agreements and pull requests | [Contributing](CONTRIBUTING.md) and [Security](SECURITY.md) |
