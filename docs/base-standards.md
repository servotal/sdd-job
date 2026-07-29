---
description: General engineering standards for this project — principles, architecture, language rules and links to area-specific standards. Applies to anyone/anything touching this codebase, human or AI. For rules specifically about how AI agents should operate in this repo (skills, symlinks, OpenSpec workflow behavior), see agents-standards.md instead.
alwaysApply: true
---

> **Conformance language**: this document is a normative contract — implementers (human or
> AI) generate code and tests directly from it. The key words **MUST**, **MUST NOT**,
> **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**,
> and **OPTIONAL** are to be interpreted as described in
> [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## 1. Core Principles

- **Small tasks, one at a time**: Work in baby steps, one at a time. Agents MUST NOT advance
  more than one step without checking in.
- **Test-Driven Development**: Implementations MUST start with a failing test for any new
  functionality (TDD), per the task's own details.
- **Type Safety**: All code MUST be fully typed — Python via type hints on every signature,
  with dataclasses (stdlib) or Pydantic v2 for structured config/state records. `mypy` SHOULD
  pass (strict where practical); the pure decision core MUST NOT use `Any` except at explicit
  boundary-parsing points.
- **Clear Naming**: Variables and functions SHOULD have clear, descriptive names.
- **Incremental Changes**: Changes SHOULD be incremental and focused over large, complex
  modifications.
- **Question Assumptions**: Implementers SHOULD question assumptions and inferences rather
  than silently proceeding on them.
- **Pattern Detection**: Repeated code patterns SHOULD be detected and flagged for
  extraction, not silently duplicated.

## 1.1 Python Job Architecture (applies to every job in this repo from day 1)

This SDD targets **python jobs**: command-line batch or long-running processes (ETL loaders,
pollers, reconciliation batches, scheduled generators, trading loops) — **not** web services,
REST APIs, or frontends. Every job MUST follow this skeleton:

- **Layered, dependencies pointing inward**: `core` (config/setup/logger/signals) → command /
  adapter layer → **pure decision core** → persisted per-target state. The pure core depends on
  nothing job-framework-specific; adapters depend on the core, never the reverse. Full detail in
  [Job Standards](./job-standards.md).
- **Decision/execution separation is the top rule**: a pure, testable core decides *what* to do
  (`decide(...) → Decision`) with no I/O; a thin execution/adapter layer performs the side
  effects. This is what makes a job unit-testable without live credentials or a live target.
- **State is mutated only from executed results**, never speculatively from a decision a later
  guard can still abort. Speculative state writes are a defect. See
  [Job Standards](./job-standards.md#state-and-persistence).
- **One or several actions**: the action set is expressed as `argparse` subcommands (one per
  target family) + mode flags (live loop / one-shot / dry-run / status), resolved to a Command
  via a factory. A job MUST support running against **one target or many** in a single
  invocation, with a deterministic, documented target-resolution order. Every declared flag
  MUST be honoured — a parsed-but-dead flag is a defect.
- **Configuration & secrets**: config is loaded per environment (`<JOB>_ENV` →
  `config.<env>.{yaml,json}`) with a deterministic search order and an explicit `--configfile`
  override; runtime *wiring* (`setup.yaml`) is kept separate from *business* config. **Secrets
  MUST come from the environment**, never from committed config files or logs.
- **Safe execution**: `--dry-run` MUST exercise the full decision path while simulating side
  effects; going live is the explicit absence of `--dry-run`. Per-target failures MUST be
  isolated (one bad target never aborts the pass), and the job MUST be safe to stop/restart at
  any point (`SIGTERM`/`SIGINT` graceful stop, `SIGHUP` reload) without double-acting.
- **Repo topology**: this repo (`sdd-job`) is the **shared SDD base** — standards, agents, and
  skills — that every job repo consumes as a **git submodule**. Job-specific artifacts
  (`data-model.md` describing that job's config/state, the job's own README, its `openspec/`
  history) MUST stay local to each job, never merged back into this repo.
- **Agent interaction pattern** (Claude-Code-inspired): agents MUST work in an **ephemeral
  workspace** and produce **curated outputs** — they MUST NOT mutate shared/production state
  directly. Any effectful action (write, publish, delete, deploy) REQUIRES **explicit
  confirmation** before it is persisted — mirrors the permission model this SDD tooling itself
  follows.

## 2. Language Standards
- **English Only**: All technical artifacts MUST use English, including:
    - Code (variables, functions, classes, comments, error messages, log messages)
    - Data schemas and database names
    - Configuration files and scripts
    - Test names and descriptions

- **English or Spanish**: For non-technical artifacts, English SHOULD be preferred, but
  Spanish MAY be used when necessary:
    - Documentation (README, guides, API docs)
    - Jira tickets (titles, descriptions, comments)
    - Git commit messages

## 3. Specific standards

For detailed standards and guidelines specific to different areas of the project, refer to:

- [Job Standards](./job-standards.md) - Python job architecture, CLI/actions, config, logging, run modes, signals, state/persistence, testing, packaging, security
- [Infrastructure Standards](./infrastructure-standards.md) - Deployment (systemd per environment, cron, containers), log rotation, secret delivery, and operational best practices for jobs
- [Development Guide](./development_guide.md) - Local environment setup, running the job, and per-environment configuration
- [Data Model](./data-model.md) - Local, per-job template describing that job's configuration and persisted-state schema
- [Documentation Standards](./documentation-standards.md) - Technical documentation structure, formatting, and maintenance guidelines, including AI standards like this document
- [OpenSpec Tasks Mandatory Steps](./openspec-tasks-mandatory-steps.md) - Required checklist and execution rules when creating or updating OpenSpec `tasks.md` files
- [Agents Standards](./agents-standards.md) - How AI agents should operate in this repo: skills auto-loading, symlink/canonical-source integrity, OpenSpec post-apply workflow rules (root `AGENTS.md` symlinks here, not to this file)

