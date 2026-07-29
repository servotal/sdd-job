# Data Model — TEMPLATE (per-job, local)

> **This is a starting template, not a real spec.** `docs/data-model.md` is **local to each
> job** — it MUST NOT live in `sdd-job` as a real model. Copy this file into your job's repo and
> replace every `<placeholder>` with the job's actual configuration and persisted-state schema,
> following the same structure. For *how* to structure the code around this data, see
> [job-standards.md](./job-standards.md).

For a python job the "data model" is two things: the **configuration** the job reads at startup,
and the **per-target state** it persists across runs. Document both here.

## 1. Configuration schema (`config.<env>.{yaml,json}`)

What the job reads per environment (selected by `<JOB>_ENV`). Secrets are **referenced** here by
name only — their values come from the environment, never this file.

| Key | Type | Required | Default | Notes |
|---|---|---|---|---|
| `<section>.<key>` | `<type>` | yes/no | `<default>` | `<what it controls>` |
| `secrets.<name>` | env-var ref | yes | — | value read from `$<ENV_VAR>`, never stored here |

- **Environments**: `<list the envs, e.g. local / staging / prod, and how they differ>`.
- **Runtime overrides** (if any): `<what may be overridden at runtime from the state dir, and the
  single allow_overrides switch that gates it>`.
- **Setup (`setup.yaml`)**: runtime wiring only — `states:` dir, `persist:` dir, etc. Kept
  separate from business config above.

## 2. Persisted state schema (`states/*.json`)

One record **per target**, keyed by target id. Loaded on start, updated **only from executed
results** (never speculatively). Model as a dataclass with explicit defaults.

### `<StateRecord>` (one per `<target>`)

| Field | Type | Default | Notes |
|---|---|---|---|
| `<target_id>` | str | — | the key |
| `last_action` | str | `"INIT"` | `<state-machine label: INIT / ... / DONE>` |
| `last_run_ts` | float | `0.0` | epoch seconds of last executed action |
| `<field>` | `<type>` | `<default>` | `<meaning>` |

- **State machine**: `<the target's lifecycle, e.g. INIT → ACTIVE → DONE, and which action drives
  each transition>`.
- **Invariants**: `<what must always hold — e.g. a counter only advances when a real result was
  recorded; aborted actions leave the record unchanged>`.
- **Compatibility**: unknown keys on load MUST be ignored; a missing field MUST deserialise to
  its default; a corrupt file MUST NOT crash the job (log, continue with a fresh record).

## 3. Actions ↔ state effects

Map each action the job can run to the state transition it causes on success. This is the
contract the decision core and the result handler share.

| Action | Precondition (guards) | State effect on executed result |
|---|---|---|
| `<action>` | `<policy/clamp/minimum checks>` | `<fields updated on confirmed result>` |
| `<report/status>` | — | none (read-only mode, no side effect) |

## Open questions (resolve before/while implementing)

- `<Anything genuinely undecided — target identity/keys, what counts as a "confirmed result",
  restart/idempotency edge cases, override safety. A real open question here is more useful than
  none.>`
