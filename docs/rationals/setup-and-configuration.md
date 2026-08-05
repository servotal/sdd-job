# Rational — Setup & Configuration strategy for a python job

Job-agnostic guidance on how a python job should resolve and load its configuration, runtime
wiring, logging and secrets at boot. This is the **pattern**; each job documents its concrete
instance locally (in its own `docs/rationals/`). It expands on
[job-standards.md](../job-standards.md) §Configuration and §Security — read those first for the
normative rules; this doc explains the *how and why*.

> **Inherited vs local.** This rational is generic and lives in `sdd-job` (shared). A job's
> concrete config doc — real env-var names, file names, domain-specific config blocks — is
> **local** to that job and MUST NOT be added here.

---

## 1. The four bootstrap pieces

A job SHOULD centralise boot in a single `init` step (called by `main()`), composing four
concerns **in dependency order** — each depends on the ones before it:

| # | Piece | Reads | Produces |
|---|---|---|---|
| 1 | **Environment** | `os.environ` | validates required vars; exits non-zero if one is missing |
| 2 | **Config** | `config.<env>.{yaml,json}` (or an explicit `--configfile`) | the business `config` mapping |
| 3 | **Setup** | `setup.yaml` | a nested runtime-wiring object (e.g. `setup.dirs.states`) |
| 4 | **Logger** | `logging.json` (dictConfig) | the configured `Logger`, refined by CLI flags |

```
Environment  →  Config  →  Setup  →  Logger  →  Command (factory) → run()
 (fail fast)   (per env)   (dirs)    (logging)
```

Rationale: fail fast on a broken environment before doing any work; load *what* to do (config)
before *where* to write (setup) before *how to report* (logging).

---

## 2. Business config — `config.<env>.{yaml,json}`

- **Environment selection.** A single variable — `<JOB>_ENV` (e.g. `MYJOB_ENV`, default
  `local`) — selects `config.<env>.*`. The same code runs every environment; only the env var
  and the loaded file differ.
- **Explicit override.** An explicit `--configfile <path>` MUST win over the env-based
  selection, loaded by extension (`.json` / `.yaml` / `.yml`).
- **Search order** (first hit wins), so the job runs from a checkout, a container, or an
  installed package without code changes:

  | # | Candidate dir | Note |
  |---|---|---|
  | 1 | `$<JOB>_CONFIG_DIR` | explicit env override |
  | 2 | `<project_root>/configs` | the usual one |
  | 3 | `<cwd>/configs` | current working dir |
  | 4 | `<package_root>/configs` | packaged fallback |

  Prefer a deterministic extension order (e.g. `.yaml` before `.json`) and **log which file was
  loaded** at startup.
- **Shape.** A config file MUST be a top-level mapping; reject a non-mapping root with a clear
  error rather than proceeding. Record the resolved `config_path`/`config_dir`/`root_dir` for
  diagnostics.
- **Secrets are referenced here by name only** — never their values (see §5).

---

## 3. Runtime setup — `setup.yaml`

Keep **runtime wiring** (directory layout: state dir, persist dir, …) separate from **business
config**. Load it into a nested object so code reads `setup.dirs.<x>`, and create the
directories if missing.

An optional YAML loader MAY support `${NAME}` interpolation with an `:default`, resolving from:
the environment, another key in the same file, or the loaded business config. Unknown tags
should be neutralised, not crash the parse. Keep this power modest — config should stay
declarative and greppable.

---

## 4. Logging — `logging.json`

Configure logging centrally via `logging.config.dictConfig` from a committed file (overridable
with a CLI flag), then let CLI flags refine it: a level flag (default conservative, e.g.
`ERROR`), a console-stream toggle, and a quiet switch. Provide a **rotating** operational file
handler. Log decisions/actions at INFO with enough context to reconstruct behaviour offline —
and **never** log secrets.

---

## 5. Secrets — environment only

- Secrets (API keys, tokens, passwords) come from **environment variables**, delivered per
  environment (a sourced `.env` locally; `EnvironmentFile=` under systemd — see
  [infrastructure-standards.md](../infrastructure-standards.md)). Config may hold at most a
  *reference* (a key name), never the secret value.
- Required vars are validated at startup so a missing one exits non-zero (supervisor
  restarts/alerts). Real per-env config files and `.env*` MUST be git-ignored.
- If a secret is ever committed, **rotate it** — deleting the file from the tree does not undo
  the exposure.

---

## 6. Runtime overrides (hot config)

A job MAY hot-reload a subset of config (e.g. the target/symbol list, per-target params) from a
file in the state dir, behind a single `allow_overrides` switch:

- Reloaded when the file's **mtime changes** and/or on **`SIGHUP`**, without restarting.
- A malformed override file is ignored (keep the last good config, warn) — it MUST NOT be able
  to push the job into an unintended side effect.

---

## 7. Precedence, end to end

Highest first:

1. `--configfile <path>` — the whole business config.
2. `<JOB>_ENV` → `config.<env>.*` — via the candidate-dir search.
3. Runtime overrides — only the keys they carry.
4. CLI flags — orthogonal behaviour (log level, `--dry-run`, safety flags, run modes).

Directories (config search and the state dir) resolve relative to the project root / cwd,
overridable with `<JOB>_CONFIG_DIR`.

---

## 8. Checklist for a new job

- [ ] `init` composes Environment → Config → Setup → Logger in that order.
- [ ] `<JOB>_ENV` selects `config.<env>.*`; `--configfile` overrides; loaded path is logged.
- [ ] `setup.yaml` holds only runtime dirs; business config is separate.
- [ ] Logging via `dictConfig` + CLI level/stream/quiet; rotating file handler; no secrets.
- [ ] Secrets from env only; required vars validated at startup; `.env*` git-ignored.
- [ ] Overrides (if any) gated by `allow_overrides`, reloaded on mtime/`SIGHUP`, fail-safe.
- [ ] The job's **concrete** config doc lives in that job's local `docs/rationals/`.
