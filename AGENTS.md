# AGENTS.md — JOSYN Platform

> Read this file first in any session touching the JOSYN platform.
> It is self-contained. No external configuration or prior session context is required.
> A fresh clone on a fresh machine is the assumed starting point.

---

## 1. Platform Identity

**JOSYN** (Job System Next) is a platform for executing scheduled jobs as isolated executable
processes. A central scheduler orchestrates execution: it spawns job processes, hands them
arguments via a named-pipe protocol, and receives results or error reports in return.

All repos are siblings under a common parent directory:

| Repo | Role | Relative path |
|------|------|---------------|
| `josyn-foundation` | Infrastructure primitives — Result pattern, serialization, IPC transport | `../josyn-foundation` |
| `josyn-jap` | JAP protocol contracts — shared contracts and logging | `../josyn-jap` |
| `josyn-job-host` | Job execution runtime — library linked by each job executable | `../josyn-job-host` |
| `josyn-backend` | Scheduler and session-orchestration layer | `../josyn-backend` |
| `josyn-commons` | Generic utility helpers — domain-agnostic, never referenced by Foundation | `../josyn-commons` |
| `josyn-platform` | Architecture, decisions, docs — **this repo** | `../josyn-platform` |

---

## 2. This Repo's Authority

`josyn-platform` is the **single source of truth** for:
- Platform architecture and cross-cutting technical decisions
- Coding standards and design principles
- Sanity check criteria and agent instructions

All other repos are **subjects** of review. They do not define architecture or standards — they
implement against the rules defined here.

---

## 3. Self-Sufficiency — Tradeoff and Governance Risk

This repo is deliberately self-sufficient: every instruction an agent needs lives here.
No external configuration, no company-wide shared instruction repo, no session cache.
A fresh clone on a bare machine is the assumed and supported starting point.

**The price of this design:**

1. **Knowledge stays private.** The coding principles, sanity criteria, and architectural
   decisions documented here benefit only JOSYN. Other company projects do not inherit them.

2. **Collision risk.** If the company introduces shared agent instructions at the org level
   (e.g., an org-wide `AGENTS.md` or a globally applied `copilot-instructions` repo), those
   instructions and the JOSYN-local ones will coexist in the same agent session. There is no
   defined precedence. JOSYN's rules are deliberately divergent from typical enterprise C#
   (no DI containers, no OOP-by-default, no thrown exceptions) — a silent override in either
   direction would go unnoticed and produce wrong behaviour.

**Current mitigation (Option C):**

JOSYN-local instructions take precedence over any company-wide or session-level defaults
for any agent operating within this repo. If a conflict is detected between a local rule
and an externally sourced instruction, the local rule wins and the conflict should be flagged.

**Long-term change path (Option B):**

The universally applicable parts of this instruction set — Result pattern discipline,
static-first design, immutability-by-default, test naming conventions, sanity check method —
are good enough to benefit other projects. The right long-term move is to extract those into
a shared company-level instruction repo and have JOSYN extend it with its platform-specific
additions. This restores the knowledge-sharing benefit while keeping the specialised rules
local. It is a deliberate project, not a note — undertake it when the company scales up its
agent infrastructure.

---

## 4. Agent Behavior

- Stay sceptical — do not be a pleaser.
- Always explain your plan before running anything.
- When in doubt about intent or scope: stop and ask.
- You are not the final decision-maker — the human always reviews.

### Confirmation gate for write operations

**In any session framed as discussion, design, or planning — and before any write operation
in any other session — the agent must:**

1. Briefly state what it is about to do and why.
2. Wait for explicit confirmation from the human before creating or editing any file,
   or running any git operation (`commit`, `push`, `branch`, `merge`, etc.).

Passing a rubber-duck critique, completing an inference phase, or reaching an internal
conclusion does not count as confirmation. Only an explicit go-ahead from the human does.

This rule has no exceptions. A plan that has not been confirmed is not a plan that may be executed.

---

## 5. Knowledge Map

No content is duplicated here. Read the canonical source when a topic requires detail.

| Topic | Canonical source |
|-------|-----------------|
| Architecture, runtime flow, component map | `architecture/overview.md` |
| Naming conventions, namespace rules, directory structure | `architecture/naming-conventions.md` |
| Repo structure patterns (Pattern A vs B, per-repo assignment) | `architecture/repo-structure-conventions.md` |
| `.local-build` purpose, characters, script conventions | `architecture/local-build.md` |
| Coding standards, principles, Result pattern reference | `architecture/coding-standards.md` |
| Storage realm: schema, DDL scripts, `IxxxRecord` convention, dev setup | `architecture/storage.md` |
| Architectural decision records | `decisions/` |
| Per-repo summaries (packages, assemblies, current state) | `repos/` |
| Sanity check protocol, inference algorithm, execution steps | `sanity/README.md` |

---

## 6. Sanity Check Protocol

The full sanity check protocol — command surface, profiles, inference algorithm, confirmation
gate, execution steps, safety contract, and fixing violations — is defined in
[`sanity/README.md`](sanity/README.md).

Read that file before running or fixing any sanity check.
