# Python Job Standards and Best Practices

> **Conformance language**: this document is a normative contract — agents generate job code
> directly from it. The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL
> NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** are to be
> interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## Table of Contents

- [Overview](#overview)
- [Technology Stack](#technology-stack)
- [Architecture Overview](#architecture-overview)
- [CLI and Actions](#cli-and-actions)
- [Configuration](#configuration)
- [Logging](#logging)
- [Execution and Run Modes](#execution-and-run-modes)
- [Signals and Lifecycle](#signals-and-lifecycle)
- [State and Persistence](#state-and-persistence)
- [SOLID and DRY Principles](#solid-and-dry-principles)
- [Coding Standards](#coding-standards)
- [Error Handling](#error-handling)
- [Testing Standards](#testing-standards)
- [Packaging and Dependencies](#packaging-and-dependencies)
- [Deployment](#deployment)
- [Security Best Practices](#security-best-practices)

## Overview

A **python job** in this repo is a long-running or batch process driven from the command line:
it loads configuration, initialises logging and runtime setup, then executes **one or several
actions** against one or more targets. It is **not** a web service and **not** a request/response
API — there is no HTTP framework, no message bus, and no frontend. The canonical reference
implementation these standards are extracted from is the Mandarina trading bot
(`mandarina_bot`), but the patterns here are **job-agnostic**: an ETL loader, a reconciliation
batch, a poller, a scheduled report generator, or a trading loop all share the same skeleton.

The job MUST be organised so that **decision logic is separated from side effects**: a pure,
testable core decides *what* to do, and a thin execution/adapter layer performs the I/O. This
is the single most important rule — it is what makes a job unit-testable without live
credentials or a live target system.

## Technology Stack

### Core

- **Language**: Python 3.10+ (3.12+ RECOMMENDED for new jobs).
- **CLI**: the standard library [`argparse`](https://docs.python.org/3/library/argparse.html) —
  global flags on the root parser, one **subcommand per action/target family**, a `command_class`
  default per subparser resolved through a factory. Jobs MUST NOT invent a bespoke argument
  parser when `argparse` subparsers express the action set.
- **Config**: YAML (preferred) or JSON, loaded per environment (see [Configuration](#configuration)).
  `PyYAML` (`yaml.safe_load`) for YAML. Config MUST NOT be executable Python.
- **Typing/validation**: type hints on every signature. Structured config/state SHOULD be
  modelled with dataclasses (stdlib) or Pydantic v2 where richer validation is warranted;
  dataclasses are the default for plain state records.
- **Logging**: stdlib `logging` configured via `logging.config.dictConfig` from a JSON/YAML file
  (see [Logging](#logging)) — never `print()` for operational output.

### Testing and Tooling

- **pytest** for all tests (`pytest-cov` for coverage). No live network in unit tests.
- **Linting/formatting**: `ruff` (lint + format). Static typing: `mypy` (strict where practical).
- **Package/env manager**: one per project, declared and used consistently — `uv` (RECOMMENDED
  for new jobs) or `pipenv`/`Pipfile` (as the reference job uses). Contributors MUST use the
  project's chosen manager, not a mix of bare `pip`, `poetry`, and `pipenv`.

## Architecture Overview

A job MUST be layered so that dependencies point **inward** — the pure core depends on nothing
job-framework-specific, and adapters depend on the core, never the reverse:

```
<job_package>/
├── __main__.py            # `python -m <job_package>` → main()
├── job.py                 # CLI (argparse), factory (subcommand → Command class), main()
├── core/
│   ├── setupjob.py        # Config, Setup, Logger, Environment
│   ├── signals.py         # SIGHUP/SIGTERM/SIGINT binding → command hooks
│   ├── logconf.py         # logging dictConfig helpers
│   └── ...                # yaml loader, connectors, misc helpers
├── <command>/             # command/adapter layer (one per target family / integration)
│   ├── base_command.py    # BaseCommand ABC: run() loop, action selection, sizing, I/O
│   └── <target>/          # concrete Command + its client/adapter wrapper
└── <logic>/               # pure decision core (no I/O)
    ├── decision.py        # decide(...) → Decision  (pure function of inputs)
    ├── <engine>.py        # step(): reads inputs, calls decide(), maps Decision → action
    ├── state.py           # persisted record dataclass(es)
    └── states.py          # JSON persistence for the records
```

**Dependency rule** (MUST hold):

- `core/` depends only on the standard library and small utilities — never on a specific
  command or logic module.
- The **decision core** (`decision.py` and the record dataclasses) MUST be **pure**: given
  inputs (current readings, current state, config) it returns a decision. It MUST NOT perform
  I/O, place orders, call APIs, read the clock as hidden global state, or mutate persisted
  state.
- The **command/adapter layer** is the only place that performs side effects (network, orders,
  writes) and is the **composition root** where config, targets, external-system constraints,
  and run-mode (live/dry-run) are combined.
- **State mutation MUST follow real execution, not intent.** Persisted state MUST be updated
  from the *result* of an executed side effect (a confirmed fill/result), never speculatively
  from a decision that a later guard can still abort. Speculative state writes are a defect:
  in the reference job they left counters that no execution ever backed, permanently wedging
  those targets. See [State and Persistence](#state-and-persistence).

## CLI and Actions

### Entrypoint

- `__main__.py` MUST expose `python -m <job_package>` and delegate to `job.main()`.
- `job.py` parses args, initialises Config/Setup/Logger/Environment, then resolves the
  concrete command via a **factory**: each subparser sets `command_class`, and `main()` calls
  `command_class.factory(args=..., config=..., setup=..., logger=...).run()`. Adding a new
  action family MUST NOT require editing a central `if/elif` dispatch — register a subparser
  with its `command_class`.

### One or several actions

A job MUST make explicit *which* action(s) it runs and over *which* target(s):

- **Action family = subcommand.** Each subcommand groups the actions for one integration/target
  family (in the reference job, one per exchange). A subcommand MUST be required unless the job
  has exactly one; if none is given, print help to stderr and exit non-zero.
- **Action mode = flags** on the subcommand (or global), e.g. a live loop (`--live`), a
  one-shot check (`--check`), a status snapshot (`-S`), a dashboard (`-D`). Modes MUST be
  documented in the job's own README and MUST be mutually coherent (a mode that only reports
  MUST NOT place side effects).
- **Single vs multiple targets.** The command layer MUST support running against **one target
  or many** in a single invocation: an explicit `--targets`/`--symbols` list, a configured set,
  or a runtime-overridable list. Target resolution order MUST be documented and deterministic
  (CLI > runtime overrides > configured set > config default, as the reference job does).
- **Every declared flag MUST do something.** A safety or mode flag that is parsed but never
  read is a defect (the reference job shipped `--no-buys`/`--no-sells`/`--check` that were
  wired into `argparse` but dead in the execution path — a live operator relying on them would
  be misled). If a flag is declared it MUST be honoured in the code path it claims to govern,
  and covered by a test.

## Configuration

- **Sources, in precedence order** (highest wins): explicit `--configfile` → environment-based
  file selection (`<JOB>_ENV` naming the environment, e.g. `config.<env>.yaml`) → discovered
  `configs/` directory. The resolution MUST be deterministic and logged at startup (which file
  was loaded, from which directory).
- **Config search dirs** SHOULD follow a documented candidate order (env override dir →
  project-level `configs/` → CWD `configs/` → package-level `configs/`), stopping at the first
  hit. Both `.yaml`/`.yml` and `.json` MUST be accepted, dispatched by extension.
- **Config file MUST be a top-level mapping.** Loaders MUST reject non-mapping roots with a
  clear error, not silently proceed.
- **Runtime overrides** (hot config): a job MAY support overriding target lists / per-target
  parameters at runtime from a JSON file in the state dir, reloaded when its `mtime` changes
  (and/or on `SIGHUP`). Overrides MUST be behind a single `allow_overrides` switch so they can
  be globally disabled, and MUST fail safe (invalid override file → keep last good config, warn).
- **Setup vs config.** Runtime *wiring* (directory layout: `states/`, `persist/`, …) lives in a
  separate `setup.yaml` loaded into a `Setup` object; **business** parameters live in
  `config.<env>.*`. Do not conflate them.
- **Secrets MUST NOT live in config files** — see [Security](#security-best-practices). Config
  MAY reference a key *name*; the value comes from the environment.

## Logging

- Logging MUST be configured centrally via `logging.config.dictConfig` from a committed
  `configs/logging.json` (or `.yaml`), not hand-built in scattered modules.
- A job MUST support at minimum: a **rotating file handler** (operational log under the working
  dir / a configured path) and an **optional console stream** toggled by a CLI flag
  (`-s/--stream`), with a global `-q/--quiet` that disables output. Log level MUST be settable
  from the CLI (`-l/--loglevel`).
- Log **decisions and actions** at INFO with enough context to reconstruct behaviour offline
  (target, action, size/parameters, reason). Diagnostic detail goes to DEBUG. Operational logs
  MUST NOT contain secrets.
- Use `%`-style lazy logging (`logger.info("x=%s", x)`), never f-strings that format eagerly at
  every call site regardless of level.

## Execution and Run Modes

- The command `run()` is the lifecycle root: create adapters, create the logic engine, bind
  signals, resolve targets, then either run **once** (one-shot modes) or enter the **live loop**
  (`while args.live and not should_stop()`), re-resolving targets each iteration so runtime
  overrides take effect, sleeping `interval_seconds` between passes.
- **Dry-run is mandatory and MUST be real.** A `--dry-run` mode MUST execute the full decision
  path and **simulate** side effects (log the intended action, synthesise a plausible result to
  feed state) without touching the live system. Going live MUST be the explicit absence of
  `--dry-run`, never the default assumption in code.
- **Per-target isolation.** One target failing (bad reading, adapter exception) MUST NOT abort
  the whole pass — catch, log with `exc_info`, continue to the next target.
- **Idempotency / guards before side effects.** Before executing, the command layer MUST apply
  its guards (target-category policy, available-balance/quantity clamps, external minimums). If
  a guard aborts the action, **no state is mutated** and the pass continues.

## Signals and Lifecycle

- The command MUST install handlers via a small signal binder:
  - `SIGHUP` → **reload** (re-read overrides/target list and let the logic react via an optional
    `on_reload()` hook) — MUST NOT tear down the process.
  - `SIGTERM` / `SIGINT` → **graceful stop**: set a `stop_event`, finish the current target if
    safe, close adapters (`broker.close()`-style), then exit. Handlers MUST NOT do heavy work
    inline; they flip state the loop checks via `should_stop()`.
- Graceful stop MUST be safe to call twice and MUST NOT leave partial side effects half-applied.

## State and Persistence

- Persisted state is **per target**, stored as JSON in the state dir (one file, keyed by
  target). The store MUST: create the state dir if missing; tolerate/ignore unknown keys when
  loading (forward/backward compatibility); preserve non-record top-level keys (meta/version);
  and never crash the job on a corrupt file (log and continue with a fresh in-memory record).
- Records SHOULD be dataclasses with explicit defaults so a missing field deserialises to a
  sane value.
- **The golden rule (repeated because the reference job violated it):** update state from the
  **confirmed result of an executed side effect**, in the result-handler (`on_fill`-style), not
  from the decision. If execution is skipped or aborted by any guard, state MUST be unchanged.
- State is the job's memory across restarts; it MUST be safe to stop and restart the job at any
  point without double-acting or losing the position/progress it had.

## SOLID and DRY Principles

- **Single Responsibility**: the decision core decides; the command executes; the adapter talks
  to the external system; `core/` wires config/log/setup. Do not blur these.
- **Open/Closed**: add a new target family via a new Command subclass + subparser, not by
  branching inside an existing command.
- **Dependency Inversion**: the command depends on an adapter *interface* (a `Protocol` /
  small wrapper), so a fake adapter can drive it in tests.
- **DRY**: cross-cutting concerns (price/reading extraction, sizing, clamps, min-size checks)
  live once in `base_command`, shared by every concrete command — not copy-pasted per target
  family.

## Coding Standards

### Naming
- `snake_case` for functions/variables/modules; `PascalCase` for classes; `UPPER_SNAKE_CASE`
  for constants — PEP 8.
- Actions/decisions are named as verbs (`decide`, `execute`, `step`); records/state as nouns.

### Type Safety
- Every function signature MUST be fully typed. The decision core MUST NOT use `Any` except at
  explicit boundary-parsing points (e.g. normalising a raw adapter dict).

## Error Handling

- Distinguish **expected operational conditions** (target skipped, minimum not met, insufficient
  balance) — logged and handled as normal flow — from **defects** (unexpected exceptions),
  which MUST be caught at the per-target boundary with `exc_info=True` and MUST NOT crash the
  loop.
- Adapter/network calls MUST be wrapped so a transient failure degrades one pass, not the whole
  job. Code MUST NOT swallow exceptions silently (`except: pass`) around side effects — at
  minimum log them.
- Fail closed on side effects: if the code cannot verify a precondition (price, balance,
  minimum), it MUST skip the action, not guess.

## Testing Standards

- **Unit tests** target the **pure decision core** with no I/O: given inputs and state, assert
  the returned `Decision`. These are the bulk of coverage and MUST run without credentials or
  network.
- **Command/adapter tests** use a **fake adapter** (in-memory) to drive `execute()`/`run()` one
  pass, asserting: correct action, guards honoured (dry-run places no real order, minimums
  respected, sells clamped to available quantity), and **state mutated only on executed
  results**.
- **AAA** structure; behavioural test names: `test_<unit>_<condition>_<expected>`.
- A green suite is a completion gate. If the reference job's own suite has known-failing tests
  (e.g. an assertion that an adapter is called exactly once while the code calls it twice),
  treat that as a real defect surfaced by the test, not as a test to relax.

## Packaging and Dependencies

- Dependencies MUST be declared in the project's chosen manifest (`pyproject.toml` for `uv`, or
  `Pipfile`/`Pipfile.lock` for pipenv) and locked. Contributors and agents MUST install/run via
  the chosen manager, not ad-hoc `pip install`.
- The job MUST be runnable as `python -m <job_package> <subcommand> [flags]` from a clean
  checkout after `install`.

## Deployment

Deployment specifics (systemd instances per environment, cron for batch jobs, container
entrypoints, log rotation, secret delivery) live in
[Infrastructure Standards](./infrastructure-standards.md). The application-level contract that
doc relies on:

- The container/service entrypoint MUST be the job invocation itself
  (`python -m <job_package> <subcommand> -L …`), one process per environment, with the
  environment selected by the `<JOB>_ENV` variable and secrets supplied via `EnvironmentFile`.
- The job MUST exit non-zero on fatal misconfiguration (missing required env var, unresolved
  config) so a supervisor can restart or alert.

## Security Best Practices

- **Secrets via environment/secret manager only.** API keys, tokens, and passwords MUST NOT be
  committed to config files or the repo, and MUST NOT be written to logs. The reference job
  shipped a live API key in `config.<env>.yaml`; a job following these standards keeps only the
  *reference* to a secret in config and reads the value from the environment. If a secret is
  ever committed, it MUST be rotated, not just deleted from the file.
- Least privilege for the credentials the job uses against its target system.
- Runtime overrides and any operator-writable control file MUST be validated before use; a
  malformed control file MUST NOT be able to make the job take an unintended side effect.
