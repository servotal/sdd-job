---
description: Claude Code-specific configuration. General project rules that apply regardless of coding agent live in AGENTS.md — read that first, this file only adds Claude-specific behavior on top.
alwaysApply: true
---

## 1. Read AGENTS.md and base-standards.md First

`AGENTS.md` (symlink to `docs/agents-standards.md`) has the agent-agnostic rules for **how any
AI agent should operate in this repo** (skills auto-loading, symlink integrity, OpenSpec
post-apply workflow). It in turn points to `docs/base-standards.md` for the **general
engineering standards** (core principles, python-job architecture, language rules, links to
job/infrastructure/documentation standards). Load and apply both before anything in this file.

## 2. Planning Model Requirement

Planning workflows must run with Opus high reasoning.

This requirement applies to:
- `enrich-us`
- `openspec-propose` (`opsx:propose`)

Before starting any of these workflows, verify the session is using Opus high reasoning. If
it is not, **self-correct** by adding `"model": "claude-opus-4-7"` to `.claude/settings.json`
(use the `update-config` skill or edit directly), then continue — do not stop and ask the
user. Do the same to come back to sonnet medium for any other step.
