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
| `josyn-adapter-contracts` | JIP protocol contracts for company adapter EXEs — the platform/company boundary layer (ADR-023) | `../josyn-adapter-contracts` |
| `josyn-job-host` | Job execution runtime — library linked by each job executable | `../josyn-job-host` |
| `josyn-session-broker` | Per-session boundary EXE — brokers between the backend world and the job developer's world (ADR-025) | `../josyn-session-broker` |
| `josyn-backend` | Scheduler and session-orchestration layer — NuGet library packages consumed by `josyn-session-broker` | `../josyn-backend` |
| `josyn-commons` | Generic utility helpers — domain-agnostic, never referenced by Foundation | `../josyn-commons` |
| `josyn-platform` | Architecture, decisions, docs — **this repo** | `../josyn-platform` |
| `josyn-playground` | Consumer playground — demos, experiments, non-platform code | `../josyn-playground` |
| `josyn-toolbox` | Maintainer tooling — deployment scripts, code generators, machine-sync utilities | `../josyn-toolbox` |
| `josyn-contoso` | Demo company adapter — implements platform extension points with fake data; not a platform component | `../josyn-contoso` |
| `josyn-docs` | Generated HTML documentation site — output target for site-builder tooling; not a source repo | `../josyn-docs` |

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

## 4. Language

**All agent responses and all documents are written in English.**

This applies regardless of the language the human uses in a given prompt. The maintainer's
primary working language is English. German (or any other language) may appear in user
messages; the agent always responds in English and produces all file content in English.

This rule may be overridden only by an explicit instruction in a concrete prompt
(e.g., "write this document in German"). That override applies to that single artefact only
and does not carry over to subsequent responses or documents.

---

## 5. Agent Behavior

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

## 6. Knowledge Map

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
| Narrative concept guides (cross-repo, visual) | `guides/` |
| Per-repo summaries (packages, assemblies, current state) | `repos/` |
| Sanity check protocol, inference algorithm, execution steps | `sanity/README.md` |
| Project status, milestones, what's done and what's next | `ROADMAP.md` |
| docs-index vocabulary (type, perspective, state closed lists) | `architecture/docs-index-vocabulary.md` |
| Master documentation index (all repos, all docs) | `docs/docs-index.json` |
| AI enrichment prompt for docs-index | `docs/enrich-docs-index-prompt.md` |

---

## 7. Sanity Check Protocol

The full sanity check protocol — command surface, profiles, inference algorithm, confirmation
gate, execution steps, safety contract, and fixing violations — is defined in
[`sanity/README.md`](sanity/README.md).

Read that file before running or fixing any sanity check.

---

## 8. NuGet Rules

- **Never increase package versions.** Version numbers are managed by the maintainer, not by
  agents. Leave all `<Version>` values exactly as found.
- **Always clear the user NuGet cache before re-packing.** Every repo and solution sub-folder
  that produces a NuGet package has a `.local-build\clean.cmd`. Run it before `pack.cmd`
  whenever a package is rebuilt, to ensure the updated `.nupkg` is picked up by consumers
  and not shadowed by a stale cache entry.

---

## 9. Code Generation Principles

> Canonical detail: `architecture/coding-standards.md` § Principle 9.

The overview and flow must be graspable at a glance. This is the dimension most often
sacrificed when code is generated. Apply these rules on every file produced or edited:

**Short methods.**
No method should be so long that its structure is not immediately apparent. If distinct
logical phases exist — parse, validate, execute, return — each phase is a candidate for
extraction into a named helper.

**Named logical groups.**
When extracting, prefer pure functions and immutable intermediates. Name for *what* the
method does, not *how* it does it.

**Nested helpers.**
If a helper only has meaning inside one super-method, place it as a nested local function
*after the `return`* in that method. The happy path stays on top; the mechanics go below.

```csharp
private static async Task<int> Main(string[] args)
{
    if (!TryParseSessionGuid(args, out var guid))
        return FailWith("Usage: ...");

    var result = await RunServer(guid);
    return result.Succeeded ? 0 : 1;

    // ── helpers ───────────────────────────────────────────────────────
    static bool TryParseSessionGuid(string[] args, out Guid guid) { ... }
    static int  FailWith(string message) { ... }
}
```

**Partial classes for logical steps.**
When a class contains many methods that cluster into lifecycle phases or responsibility
areas, split them into partial class files with a `.`-separated naming pattern:

```
Host.Entrypoint.cs    ← startup, arg parsing, bootstrap
Host.Adapters.cs      ← adapter spawn and lifecycle
Host.Prepare.cs       ← per-session preparation phase
Host.Negotiation.cs   ← accept/reject handshake
```

Each file owns one coherent concern. The full picture emerges from the file listing alone.

**No mutation of reference-type parameters.**
Never modify a reference-type object that was passed into a method as a parameter.
Produce and return a new instance instead. Principle 2 (immutability) applies at every
method boundary — not just to field and property declarations. Mutating an incoming
reference silently violates the caller's contract even when the type itself allows it.

**Comments.**
A short comment on anything not crystal-clear is more than appreciated. Explain *why*, not
*what*. A comment that restates the code adds noise; one that explains a non-obvious
constraint or tradeoff saves the next reader from reconstructing the reasoning from scratch.
