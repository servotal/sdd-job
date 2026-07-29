# Infrastructure Standards

> **Conformance language**: this document is a normative contract (deployment topology,
> operational gates). The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL
> NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** are to be
> interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

Deployment and operational standards for **python jobs** (see
[Job Standards](./job-standards.md) for the application-level architecture these jobs run).
A python job is a single process invoked as `python -m <job_package> <subcommand> [flags]`;
it has no HTTP surface, no cluster of services, and no frontend. Deployment is therefore about
**running one process per environment reliably**, feeding it config and secrets, keeping it
alive (or scheduling it), and rotating its logs — not about orchestration.

## Deployment shapes

Pick the shape that matches how the job runs; a job MAY use more than one (e.g. a live loop as a
service plus a periodic maintenance run via cron).

- **Long-running loop → a supervised service** (systemd, RECOMMENDED). The supervisor restarts
  the process on crash and starts it on boot.
- **Periodic batch → cron / systemd timer**. One-shot invocation on a schedule; the job runs a
  single pass and exits.
- **Container** (Docker/Podman/Kubernetes `CronJob` or `Deployment`) where the platform
  requires it — the container's **entrypoint MUST be the job invocation itself**, one process
  per container, environment selected by `<JOB>_ENV`, secrets injected as env vars.

Whatever the shape, these MUST hold:

- **One process per environment.** The environment is selected by the `<JOB>_ENV` variable; the
  same image/checkout runs every environment, differing only by env vars and the loaded
  `config.<env>.*`.
- **Non-zero exit on fatal misconfiguration** (missing required env var, unresolved config) so
  the supervisor restarts or alerts instead of looping on a broken start.
- **Secrets via the environment only** (see [Secrets](#secrets)).
- **Idempotent restart.** Because state is persisted per target and mutated only on executed
  results (see [Job Standards](./job-standards.md#state-and-persistence)), the job MUST be safe
  to stop and restart at any time without double-acting or losing progress.

## systemd (long-running loop)

Use a **template unit** (`<job>@.service`) so one unit definition serves every environment,
instantiated as `<job>@<env>`. The environment name becomes `<JOB>_ENV` and selects both the
`EnvironmentFile` (secrets) and the config file.

```ini
# /etc/systemd/system/<job>@.service
[Unit]
Description=<Job> (%i)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/opt/<job>
Environment="PYTHONPATH=/opt/<job>"
Environment="<JOB>_ENV=%i"
# Secrets for this environment, root-owned, chmod 600 — NOT in the repo:
EnvironmentFile=-/etc/<job>/%i.env
ExecStart=/opt/<job>/.venv/bin/python -m <job_package> %i --live --interval-seconds 15
# Graceful stop: the job traps SIGTERM and shuts down cleanly (see Job Standards §Signals)
KillSignal=SIGTERM
TimeoutStopSec=30
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now <job>@prod
sudo systemctl enable --now <job>@staging
# Reload target list / overrides without a restart (SIGHUP):
sudo systemctl reload <job>@prod      # if the unit maps reload → SIGHUP, or:
sudo kill -HUP "$(systemctl show -p MainPID --value <job>@prod)"
journalctl -u <job>@prod -f
```

- `Restart=always` + `RestartSec` MUST be set for loop jobs so a crash self-heals.
- The unit MUST send **`SIGTERM`** for stop (the job's graceful-stop handler) and give it a
  `TimeoutStopSec` grace window before `SIGKILL`.
- Run as a **dedicated, non-root service user** with least privilege where the platform allows;
  the example above omits `User=` for brevity — set it in real deployments.

## Scheduled batch (cron / systemd timer)

For one-shot passes, schedule the invocation directly; do not build a bespoke scheduler into the
job:

```cron
# /etc/cron.d/<job>  — run a single reconciliation pass hourly
0 * * * *  <svcuser>  <JOB>_ENV=prod /opt/<job>/.venv/bin/python -m <job_package> prod --check >> /var/log/<job>/cron.log 2>&1
```

- A batch invocation MUST exit non-zero on failure so cron/the timer surfaces it (mail/alert).
- Overlapping runs MUST be prevented (e.g. `flock`) if a pass can outlast its interval.

## Configuration & environment selection

- `<JOB>_ENV` selects the environment everywhere (systemd `%i`, cron var, container env). The
  loaded config file MUST be logged at startup (see
  [Job Standards](./job-standards.md#configuration)).
- Business config (`config.<env>.*`), runtime wiring (`setup.yaml`), and logging config
  (`logging.json`) are deployed alongside the code; **secrets are not** — they arrive via env.
- Per-environment differences MUST be expressed as config/env, never as code branches on the
  environment name.

## Secrets

- **Environment variables only**, delivered per environment via the supervisor:
  systemd `EnvironmentFile=/etc/<job>/<env>.env` (root-owned, `chmod 600`), the container
  platform's secret mechanism, or the CI/CD secret store — **never committed to the repo or any
  config file, and never written to logs.**
- Secrets are **per environment**; a value valid in one environment MUST NOT be reused or synced
  to another.
- If a secret is ever committed, it MUST be **rotated**, not merely deleted from history. (The
  reference job shipped a live API key in a committed config file — exactly the failure mode
  this rule exists to prevent.)

## Logging & rotation

- The job writes a rotating operational log (see
  [Job Standards](./job-standards.md#logging)). Under systemd, stdout/stderr also land in the
  journal (`journalctl -u <job>@<env>`).
- Log rotation MUST be configured — either the job's own `RotatingFileHandler` (size-based, as
  the reference job uses) or `logrotate` for any plain file it writes — so disk cannot fill.
- Logs MUST NOT contain secrets.

## CI/CD

- CI SHOULD run `ruff`, `mypy`, and `pytest` on every change; a green suite is the pre-merge
  gate (see [Job Standards](./job-standards.md#testing-standards)). These gates are the
  implementer's responsibility unless a pipeline is wired to enforce them.
- Deployment (copying the new checkout/image to the host, `systemctl restart <job>@<env>`, or
  rolling the container) is an **effectful, individually-confirmed action** — an agent MUST NOT
  deploy unattended.
- A deploy SHOULD go to a non-production environment first (`<job>@staging`) and be observed
  before promoting to prod.

## Observability

- **Health = the process is up and progressing.** For loop jobs, `systemctl status <job>@<env>`
  + recent journal activity is the baseline; a job MAY emit a heartbeat log line per pass so a
  stalled loop is visible.
- Metrics are OPTIONAL for a job; if exported, prefer a simple textfile/exporter the host's
  monitoring already scrapes rather than adding a server to the process.
- Operational logs MUST be greppable by target id and by action, so a past decision can be
  reconstructed offline.
