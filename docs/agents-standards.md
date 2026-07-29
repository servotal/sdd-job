---
description: Rules specifically about how AI agents/copilots should operate in this repo (skills, symlink integrity, OpenSpec workflow behavior) — not general engineering standards, see base-standards.md for those. Root AGENTS.md symlinks to this file.
alwaysApply: true
---

> **Conformance language**: this document is a normative contract that directly governs agent
> behavior in this repo. The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**,
> **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** are to
> be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

Read [base-standards.md](./base-standards.md) first — it has the general engineering
principles, platform architecture, language rules, and links to area-specific standards.
This document only covers rules about **how an AI agent working in this repo MUST behave**,
regardless of which agent it is.

## 1. Project Skills

- Skills live in `ai-specs/skills`.
- When a request matches a skill, agents MUST load and follow the corresponding `SKILL.md`
  automatically before continuing.
- Agents MUST also load any referenced files in the skill folder (for example,
  `references/*.md`) when the skill requires them.

## 2. Symlink Integrity and Agent Portability

- **Canonical Source**: Reusable artifacts MUST live in `ai-specs` as the canonical source.
  Agent-specific paths (today only `.claude`, since Claude is the only configured agent — add
  more, e.g. `.cursor`, as those agents are actually adopted; see `README.md`
  §"Multi-agent: how to add another copilot" in `sdd-job`) SHOULD reference them through
  symlinks where the target agent supports it.
- **Update Safety**: Whenever a file is renamed, moved, or its suffix changes, agents MUST
  verify and update all symlinks that target it before considering the change complete.
- **New Artifact Linking**: Whenever creating a new artifact that requires agent exposure (for
  example new agents or skills in `ai-specs`), agents MUST create the corresponding symlinks
  from the expected agent-specific reference paths.
- **External Customization Review**: Whenever customization is introduced outside `ai-specs`,
  agents MUST evaluate whether it should be moved into `ai-specs` and replaced with symlinks
  from the original locations.
- **Completion Gate**: A change MUST NOT be considered complete if it leaves broken symlinks,
  stale targets, or duplicated canonical artifacts across agent-specific folders.

## 3. Mandatory OpenSpec Artifact Updates for Post-Apply Changes

When a new fix/change request appears after `opsx:apply` (or `/apply`) and before
`opsx:archive` (or `/archive`), agents MUST treat it as a spec update first, not as an
informal "fix this quickly" — this is the core OpenSpec principle: documentation is the
source of truth.

Required order:

1. Agents MUST update the current OpenSpec change artifacts that are affected (for example:
   scenarios, requirements/specs, and `tasks.md`) — new tasks MUST be added as part of the
   initial design, in the proper section, not filed as "bugfixes".
2. If artifact regeneration is needed, agents MUST run the corresponding OpenSpec step
   (`opsx:propose` to update the change's proposal/design/tasks, or equivalent) before coding.
3. Agents MUST implement code only after artifacts reflect the new request.
4. Agents MUST re-run verification against the updated artifacts before archiving.

Agents MUST NOT apply direct code-only fixes in this window without updating OpenSpec
artifacts first.
