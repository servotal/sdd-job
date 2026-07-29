---
name: show-spec-working
description: Use when the user asks "show me X", "demo X", "walk me through X", "how X works" or requests a live feature demonstration from a spec, feature or ticket.
author: LIDR.co
version: 1.0.0
---

# show-spec-working Skill

Demonstrate a spec in a runnable way, for a **python job** (CLI batch / long-running process —
see `docs/job-standards.md`). The demonstration is a **live run of the job in a safe mode**
(`--dry-run` or a read-only mode like `-S`/status), showing the decision log and the resulting
persisted state — not a browser walkthrough and not HTTP calls.

If the user does not provide explicit context, use the spec/change currently being worked on in
this session.

Always end by reporting completion in chat.

## Trigger phrases (high priority)

Treat these expressions as execution commands, not analysis requests:

- `show me X`
- `demo X`
- `walk me through X`
- `show X working`
- `how X works`
- `prove X works`

When any of these appear, run the demonstration workflow directly.
Do not stop at a feature summary or quick report.

## Inputs

- Optional spec context from user:
  - Direct ticket id in text (for example: `SCRUM-10`)
  - Feature name
  - Subcommand / action / run mode (e.g. `--check`, `--live`, `-S`)
  - Target(s) to demo against (e.g. a specific symbol/item)
- If missing, infer from current session context and currently active work.

## Workflow

### Step 1 - Resolve target spec and scope

1. Identify the target spec/change:
   - Prefer explicit user-provided context.
   - If user text contains a ticket id pattern like `[A-Z]+-[0-9]+`, use it as primary context
     (example: `show me SCRUM-10`).
   - Otherwise, infer the spec currently being worked on.
2. Determine what the change touches:
   - The **decision core** (a `decide()` rule / condition), and/or
   - A **run mode or flag** (a new `--check`, a changed `--live` behavior, a status view), and/or
   - **Guards / state** (a clamp, a minimum, a category policy, a state transition).
3. List concrete scenarios to demo from the spec acceptance criteria, and pick the target(s)
   and inputs that will actually exercise each one.

### Step 1.1 - Anti-report guardrail

Before continuing, enforce this rule:

- Never finish after only analyzing requirements.
- Never return only a quick report when the user asked to "show" or "demo".
- If execution is blocked, explicitly report the blocker and ask for exactly what is needed to
  continue the live demo.

### Step 2 - Safe-run demonstration path

Run the job itself, in a mode that performs **no real side effects**.

1. **Prepare**: select a non-production environment (`<JOB>_ENV`), confirm required env vars are
   present (secrets from env, never config), and note the pre-run state for the target(s) you
   will demo (copy the relevant `states/*.json` entries / key fields so a diff is possible).
2. **Run one pass in a safe mode**, logging visible, e.g.:
   - `python -m <job_package> -l DEBUG -s --dry-run <subcommand> --check` (single pass), or
   - `python -m <job_package> <subcommand> -S` (read-only status), or
   - `--live --dry-run` for one interval if a loop is what the spec describes.
   (Prefix with `uv run` / `pipenv run` as the job requires.)
3. **Walk the decision log, one scenario at a time.** For each acceptance criterion, point at
   the log line(s) showing the target, the action taken, and the **reason**, and confirm it
   matches the spec.
4. **Show the guard/edge paths** the change involves: trigger a skip (minimum not met,
   insufficient balance/quantity, policy block) and show the log proving the action was skipped
   **and the state left unchanged**.
5. **Show the state effect**: diff the target's `states/*.json` before vs after and confirm it
   changed only as the change intends — and that no target advanced without an executed action
   (the speculative-mutation check).
6. **Restore**: remove/restore any scratch state or config used for the demo so repeated demos
   stay deterministic.

### Step 3 - When it can't be run safely

If a scenario genuinely cannot be shown without a real side effect, do **not** run it live.
Instead demonstrate it via its unit test(s) on the pure decision core (run the specific
`pytest -k <case>` and show the assertion), and say explicitly why a live run was not safe.

## Safety requirements

- **All live runs MUST be `--dry-run` or a read-only mode.** Never place a real order / real
  side effect during a demo.
- Mask sensitive values in chat output; never echo secrets.
- Keep runs idempotent — restore any scratch state afterward.

## Completion contract

Always send a final chat message containing:

1. Target spec/change demonstrated.
2. What was executed:
   - Command(s) run (mode + subcommand + flags).
   - Scenarios shown, with the decision-log evidence per scenario.
3. Verification result per demonstrated scenario (pass/fail with short note).
4. State restore status (if applicable).
5. Final handoff: "Demo complete — say the word to run another scenario or a different mode."

## Output format

Use this concise structure in the final chat response:

```markdown
Spec demo completed for: <spec/change>

Command(s):
- <mode + subcommand + flags>

Decision-log walkthrough:
- <scenario → target/action/reason evidence → pass/fail>

State effect:
- <target: field before → after; no target advanced without an executed action>

State restore:
- <restored / not needed / failed + reason>

Next:
- Ask me to run another scenario, mode, or target.
```
