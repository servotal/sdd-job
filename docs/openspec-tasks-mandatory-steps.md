---
description: Enforce mandatory steps from openspec/config.yaml when creating tasks.md artifacts and ensure the agent executes all manual verification for python jobs
alwaysApply: true
---

# OpenSpec Tasks: Mandatory Steps Enforcement

> **Conformance language**: this document is a normative contract enforced on every OpenSpec
> `tasks.md` artifact. The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**,
> **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** are to
> be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

These steps target **python jobs** (CLI batch / long-running processes — see
[job-standards.md](./job-standards.md)). Verification for a job is **not** curl/HTTP or browser
E2E — it is running the job in **`--dry-run`/one-shot** and asserting decisions and persisted
state. When creating or updating `tasks.md` artifacts in OpenSpec changes, you MUST:

## 1. Read openspec/config.yaml First

**BEFORE** creating or updating any `tasks.md` file, you MUST read `openspec/config.yaml` to
understand:
- Job-specific mandatory steps
- Branch naming conventions
- Task structure requirements
- Testing and documentation requirements

## 2. Mandatory Steps

All implementation tasks MUST include these steps in the correct order:

### Step 0: Create Feature Branch (MUST BE FIRST)
- **Location**: MUST be the very first step (Step 0)
- **Branch naming**: `feature/[ticket-id]` or `feature/[change-name]`
- **Action**: Create and switch to the feature branch before any code changes

### Mandatory Steps (MUST Be Included):
- **Step N**: Review and Update Existing Unit Tests (MANDATORY)
- **Step N+1**: Run Unit Tests and Verify Persisted State (MANDATORY) — **AGENT MUST EXECUTE**
- **Step N+2**: Manual Job Run — Dry-Run / One-Shot (MANDATORY) — **AGENT MUST EXECUTE**
- **Step N+3**: Update Technical Documentation (MANDATORY)

## 3. Manual Verification Requirements — CRITICAL: Agent Must Execute

**IMPORTANT**: The coding agent (AI) MUST perform all manual verification itself. It MUST NOT
delegate to the user. These steps MUST be executed by the agent before a task can be marked
completed in `tasks.md`. **The agent MUST NOT place real side effects during verification — all
manual runs use `--dry-run` (or a read-only mode such as `-S`/status).**

### Step N+1: Run Unit Tests and Verify Persisted State (MANDATORY)

**Agent Responsibility**: The agent MUST execute the tests, validate the job's persisted state
(the `states/*.json` records) before/after, and produce a report artifact in the change spec
folder. This is NOT optional and cannot be delegated.

**Implementation Steps** (Agent MUST perform):
1. **Prepare Test Environment**:
   - Ensure dependencies are installed via the job's package manager (`uv`/`pipenv`).
   - Capture the pre-run state relevant to the change (copy the relevant `states/*.json`
     entries, or their key fields, so a diff is possible).
   - Document the exact test command(s) that will be executed.
2. **Run Targeted Unit Tests First**:
   - Execute focused tests for the modified module(s) — especially the **pure decision core**.
   - Confirm prior failures are resolved and no new regressions appear in the targeted scope.
3. **Run Broader Unit Test Suite**:
   - Execute the project/unit suite required by `openspec/config.yaml`.
   - Record total counts, failures, runtime, and any flaky behavior.
4. **Verify Post-Run State**:
   - Re-check the same state indicators captured before.
   - Confirm state changed **only** as the change intends, and that no target was mutated by a
     path that did not actually execute an action (the speculative-mutation anti-pattern).
   - If a scratch/test state file was used, restore/remove it.
5. **Create Verification Report in Spec Folder**:
   - Save under `specs/<change-name>/reports/`.
   - Filename: `YYYY-MM-DD-step-N+1-unit-test-and-state-verification.md`.
   - Include executed commands, summarized results, and the state pre/post comparison.
6. **Mark Task as Completed**: Only after tests pass (or documented exceptions), state is
   verified, and the report exists.

**Report Template** (store in `specs/<change-name>/reports/`):
```markdown
# Step N+1 Report — Unit Tests and State Verification

- Date: YYYY-MM-DD
- Change: <change-name>
- Agent: <agent-name>

## Commands Executed
- `<command 1>`
- `<command 2>`

## Unit Test Results
- Targeted tests: X passed, Y failed, Z skipped
- Full/required suite: X passed, Y failed, Z skipped
- Runtime: <duration>
- Notes: <flaky tests, retries, exceptions>

## Persisted State Verification
- Pre-run baseline (states/*.json):
  - <target/field>: <value>
- Post-run validation:
  - <target/field>: <value>
- Only-on-executed-result check: <confirmed no target advanced without an action>
- Scratch state restored/removed: Yes/No
```

**Notes**:
- **The agent MUST execute tests itself** — it MUST NOT ask the user to run them.
- Mandatory even when code changes look small.
- **Task completion in tasks.md can only be marked after the report exists.**

### Step N+2: Manual Job Run — Dry-Run / One-Shot (MANDATORY)

**Agent Responsibility**: The agent MUST run the job end-to-end in a **safe mode** and verify
its behavior. This is NOT optional and cannot be delegated.

**Implementation Steps** (Agent MUST perform):
1. **Prepare**:
   - Select a non-production environment (`<JOB>_ENV`) and confirm required env vars are set
     (no secrets in config).
   - Note the pre-run state for any target(s) the change touches.
2. **Run a Single Pass in Dry-Run**:
   - Execute one pass with logging visible, e.g.
     `python -m <job_package> -l DEBUG -s --dry-run <subcommand> --check`
     (or the job's one-shot/`--live`-with-`--dry-run` equivalent).
   - Verify the **decision log**: for each target, the action and its reason are what the change
     intends.
   - Confirm **no real side effect** occurred (dry-run simulated the action) and that simulated
     results updated state as designed.
3. **Exercise the Relevant Mode(s)**:
   - If the change adds/affects a run mode or flag, run that mode and confirm the flag is
     actually honoured (a declared-but-dead flag is a defect).
   - If a read-only mode exists (`-S`/status), run it and confirm it reports correctly and
     performs no side effect.
4. **Verify Guards and Edge Cases**:
   - Trigger the guard paths the change involves (minimum not met, insufficient balance/quantity,
     policy block) and confirm the action is skipped **and state is unchanged**.
5. **Restore**:
   - Remove/restore any scratch state or config used for the run.
6. **Mark Task as Completed**: Only after the dry-run behaves correctly and state is verified.

**Notes**:
- **The agent MUST run the job itself** — it MUST NOT ask the user to run it.
- **All manual runs MUST be `--dry-run` or a read-only mode** — verification MUST NOT place real
  orders/side effects.
- Document the commands, the decision log excerpts, and the state pre/post in a report in the
  spec folder with proper naming.
- **Task completion in tasks.md can only be marked after the dry-run verification succeeds.**

## 4. Verification Checklist

Before finalizing any `tasks.md` file, verify:
- [ ] Step 0 (Create Feature Branch) is the FIRST step
- [ ] All mandatory steps from config.yaml are included
- [ ] Steps are numbered sequentially
- [ ] Mandatory steps are clearly marked with "(MANDATORY)" label
- [ ] Branch naming follows the convention: `feature/[name]`
- [ ] Step N+1 includes report path and naming in `specs/<change-name>/reports/`
- [ ] Manual steps explicitly state "AGENT MUST EXECUTE" and use `--dry-run`/read-only modes
- [ ] Tasks include state pre/post verification (and the only-on-executed-result check)

## 5. When This Applies

This rule applies when:
- Creating `tasks.md` via `/opsx:propose` or the `openspec-propose` skill
- Updating existing `tasks.md` files
- Any task creation that changes the job's code or behavior
- Implementing tasks from `tasks.md` via `/opsx:apply` or the `openspec-apply-change` skill —
  the agent MUST execute the manual verification

## 6. Example Structure

```markdown
## 0. Setup: Create Feature Branch (MANDATORY - FIRST STEP)

- [ ] 0.1 Create feature branch `feature/add-check-mode` from main/master
- [ ] 0.2 Verify branch creation and current branch status

## 1. Decision Core: Tests (TDD)
...

## 8. Review and Update Existing Unit Tests (MANDATORY)
...

## 9. Run Unit Tests and Verify Persisted State (MANDATORY - AGENT MUST EXECUTE)
- [ ] 9.1 Capture pre-run baseline for impacted targets in states/*.json
- [ ] 9.2 Run targeted unit tests for the decision core / changed modules
- [ ] 9.3 Run the required broader unit suite from config
- [ ] 9.4 Verify post-run state; confirm no target advanced without an executed action
- [ ] 9.5 Create report `specs/<change-name>/reports/YYYY-MM-DD-step-N+1-unit-test-and-state-verification.md`
- [ ] 9.6 Mark complete only after tests pass and report exists

## 10. Manual Job Run — Dry-Run / One-Shot (MANDATORY - AGENT MUST EXECUTE)
- [ ] 10.1 Set <JOB>_ENV to a non-prod env; confirm required env vars present
- [ ] 10.2 Run one pass with `--dry-run` and inspect the decision log per target
- [ ] 10.3 Confirm no real side effect; simulated results updated state as designed
- [ ] 10.4 Exercise the affected mode/flag and confirm it is honoured
- [ ] 10.5 Trigger the relevant guard path; confirm action skipped and state unchanged
- [ ] 10.6 Document commands, decision-log excerpts, and state pre/post

## 16. Update Technical Documentation (MANDATORY)
...
```

## 7. Agent Execution Requirements

**CRITICAL**: When implementing tasks from `tasks.md` (via `openspec-apply-change` or
`/opsx:apply`), the agent MUST:

1. **Execute All Manual Verification**: The agent MUST NOT ask the user to run the job. The
   agent MUST:
   - Install/prepare the environment as needed
   - Run the job in `--dry-run`/read-only mode itself
   - Verify the decision log, guard behavior, and persisted state
   - Restore any scratch state after the run
2. **Mark Tasks as Completed** only AFTER:
   - The agent has run the tests and the dry-run verification
   - All results have been verified (including the only-on-executed-result state check)
   - Scratch state has been restored
   - Outcomes have been documented
3. **MUST NOT Delegate Verification**: the agent MUST NOT ask the user to run the job or the
   tests, MUST NOT mark tasks complete without executing them, and MUST NOT place real side
   effects during verification.
4. **Document Execution**: commands run, decision-log excerpts, state pre/post, guard checks,
   and any issues + resolutions.

## Failure to Follow

If you create tasks without these mandatory steps, the user will need to manually fix
`tasks.md`. Always read `openspec/config.yaml` first and include all mandatory steps.

**If you implement tasks without running the tests and the dry-run verification yourself, you
are violating this rule. The agent MUST execute all verification to mark tasks as completed.**
