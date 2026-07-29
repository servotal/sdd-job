# Development Guide

Shared setup instructions for any **python job** in this repo (CLI batch / long-running
process — see [job-standards.md](./job-standards.md)). Job-specific deviations belong in that
job's own `docs/` (never edited here).

## Prerequisites

- **Python 3.10+** (3.12+ RECOMMENDED for new jobs)
- The job's chosen **package/env manager**, used consistently — one of:
  - **[uv](https://docs.astral.sh/uv/)** (RECOMMENDED for new jobs), or
  - **pipenv** (`Pipfile`/`Pipfile.lock`), as the reference job uses.

  Do not mix bare `pip`, `poetry`, and `pipenv` in the same job.
- **Git**

There is **no** database, message broker, container cluster, or frontend toolchain to stand up:
a python job reads config, logs, and talks to its target system through an adapter. Local
dependencies (if any) are the job's own — documented in that job's README.

## Setup

### 1. Clone and install dependencies

```bash
git clone --recurse-submodules <job-repo-url>
cd <job>

# uv:
uv sync

# or pipenv:
pipenv install --dev
```

### 2. Environment variables

A job selects its environment with `<JOB>_ENV` (e.g. `MANDARINA_ENV=local`) and reads
**secrets from the environment only** — never from committed config. Keep them in a local,
untracked env file:

```bash
# .env (local only, never committed) — source it before running
export MANDARINA_ENV=local
export <JOB>_API_KEY=...        # secret value, referenced by name in config, valued here
export <JOB>_API_SECRET=...
```

Required env vars are validated at startup; the job MUST exit non-zero if one is missing.

### 3. Configuration

Business config lives in `configs/config.<env>.{yaml,json}`, selected by `<JOB>_ENV` (or
overridden with `--configfile`). Runtime *wiring* (state/persist dirs) lives in
`configs/setup.yaml`; logging in `configs/logging.json`. See
[data-model.md](./data-model.md) for the config/state schema and
[job-standards.md](./job-standards.md#configuration) for the resolution order.

### 4. Run the job

Always validate with **`--dry-run`** first — it exercises the full decision path but performs
no live side effects:

```bash
# Dry-run a single pass at DEBUG, streaming to console:
python -m <job_package> -l DEBUG -s --dry-run <subcommand> --check

# One-shot status snapshot:
python -m <job_package> <subcommand> -S

# Live loop (real side effects) — only after dry-run looks right:
python -m <job_package> -l INFO <subcommand> --live --interval-seconds 15
```

(With `uv`, prefix `uv run`; with pipenv, `pipenv run`.)

## Testing

```bash
# uv (substitute pipenv run / plain python as appropriate):
uv run ruff check .
uv run mypy .
uv run pytest                    # full suite
uv run pytest -q test            # the reference job keeps tests under test/
uv run pytest --cov              # with coverage
```

- Unit tests target the **pure decision core** — no network, no credentials.
- Command/adapter tests drive one pass with a **fake adapter** (dry-run places nothing, guards
  honoured, state mutated only on executed results). See
  [job-standards.md](./job-standards.md#testing-standards).

## Common issues

- **`<JOB>_ENV` not set / required env var missing**: the job exits non-zero at startup by
  design — export the variable (step 2), don't hard-code it in config.
- **Config file not found**: check the search order (`--configfile` > `<JOB>_ENV` file in
  `configs/` > discovered `configs/`) — the loaded file is logged at startup.
- **A declared CLI flag seems to do nothing**: that's a defect, not a quirk — every declared
  flag MUST be honoured in the path it governs (see job-standards.md §CLI and Actions). File it.
- **A target is "stuck"** (a counter advanced but no action ran): the classic
  speculative-state-mutation bug — state must be updated only from executed results
  (see job-standards.md §State and Persistence).
- **Secret ended up in a config file or log**: rotate it, then move the value to the
  environment and reference it by name in config.
