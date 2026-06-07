# docs-index Vocabulary

**Status:** Settled
**Owned by:** josyn-platform (single source of truth for all repos)

---

## Purpose

Closed vocabularies for the three semantic axes of the docs-index.
Every value in `docs-index.json` for `type`, `perspective`, and `state`
must come from these lists — no free-form values.

Both the AI enrichment prompt and any future tag-aware tooling reference this file.

---

## `type` — What kind of document is this?

| Value | Meaning |
|-------|---------|
| `architecture-overview` | High-level, cross-cutting architecture — the big picture |
| `architecture-detail` | Deep dive into a specific architectural area |
| `component-architecture` | Architecture of a specific component or repo |
| `solution-architecture` | End-to-end solution architecture (the full system story) |
| `decision` | Architectural decision record (ADR) or settled proposal |
| `reference` | Coding standards, naming conventions, rules — normative |
| `manual` | Operational or how-to guide — procedural |
| `working-note` | Thinking, brainstorming, not settled — raw ideas |
| `overview` | README-style entry point or index page |
| `repo-summary` | Per-repo summary: packages, assemblies, current state |
| `plan` | Planning document — temporary, tied to a work item |
| `agent-instruction` | Document written for AI agents (AGENTS.md and equivalents) |

---

## `perspective` — Who is this for?

A document may serve more than one perspective. `perspective` is an array.

| Value | Meaning |
|-------|---------|
| `maintainer-architect` | The maintainer wearing the technical architecture hat |
| `maintainer-developer` | The maintainer writing or reviewing code |
| `maintainer-operator` | The maintainer running, deploying, or monitoring the system |
| `job-developer` | A developer writing a job that runs on the platform |
| `agent` | AI agents operating in the codebase |
| `stakeholder` | Manager, customer, or external interested party |

---

## `state` — How much should the reader trust this?

| Value | Meaning |
|-------|---------|
| `thinking` | Raw, unreviewed — ideas in progress, may be contradictory |
| `in-progress` | Being actively worked on — not yet stable |
| `settled` | Stable and reliable — can be acted upon |
| `superseded` | Replaced by a newer document — kept for historical context |

---

## Rules

1. Pick the **single best** `type` — no arrays, no compound types.
2. `perspective` is an array — assign every perspective that genuinely applies.
3. `state` reflects the **current trust level** of the content, not the intent.
4. When in doubt between `thinking` and `in-progress`: if it has been acted on,
   it is `in-progress`; if it is still exploratory, it is `thinking`.
5. Do not add new values without updating this file.
   This file is the schema — not the enrichment prompt, not the index.
