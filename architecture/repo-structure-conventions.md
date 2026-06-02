# Repo Structure Conventions

Two structural patterns govern how solutions and projects are arranged within a JOSYN git
repository. Every repo falls into exactly one pattern. The choice is determined by the
**nature of what the repo contains**, not by preference.

---

## Pattern A — Single-Solution Repo

Use when a repo contains **exactly one cohesive deliverable**: one library, one executable,
or one tightly coupled library + test pair that are always versioned and released together.

The git root **is** the solution root. There is no intermediate folder level.

```
<repo-root>/
├── <AssemblyName>.slnx           ← the one solution
├── Directory.Build.props         ← build output config (scoped to this solution)
├── nuget.config                  ← local package feed (scoped to this solution)
├── README.md
├── .local-build/                 ← local developer tooling (see local-build.md)
├── <AssemblyName>/               ← main project
└── <AssemblyName>.Test/          ← test project
```

`Directory.Build.props` and `nuget.config` are **repo-scoped and solution-scoped
simultaneously** — there is no distinction because there is only one solution.

**Canonical example:** `josyn-job-host`

---

## Pattern B — Multi-Solution Repo

Use when a repo contains **multiple independently releasable deliverables** that share a
common domain or layer but are versioned, packaged, and consumed separately.

The git root is a **meta-container**. It holds no solution, no build config, no feed config.
Inside it are kebab-case sub-folders — one per solution — each of which is a fully
self-contained Pattern-A root.

```
<repo-root>/
├── README.md                     ← repo-level overview only
├── LICENCE
├── .local-build/                 ← local developer tooling — batch level (see local-build.md)
├── <kebab-solution-a>/           ← self-contained solution root (Pattern A inside)
│   ├── <AssemblyName>.slnx
│   ├── Directory.Build.props
│   ├── nuget.config
│   ├── README.md
│   ├── .local-build/             ← local developer tooling — single-target (see local-build.md)
│   ├── <AssemblyName>/
│   └── <AssemblyName>.Test/
└── <kebab-solution-b>/           ← independent; same structure
    ├── <AssemblyName>.slnx
    ├── Directory.Build.props
    ├── nuget.config
    ├── README.md
    ├── .local-build/             ← local developer tooling — single-target (see local-build.md)
    ├── <AssemblyName>/
    └── <AssemblyName>.Test/
```

`Directory.Build.props` and `nuget.config` belong to the **sub-folder**, never the repo root.
Each solution controls its own build paths and feed config independently.

Sub-folder names follow the same kebab-case pattern as repo names:
`<repo-name>-<solution-descriptor>` (e.g., `josyn-foundation-result-pattern`).

### `.local-build/` at two levels

Pattern B repos carry `.local-build/` at two distinct levels with different characters —
**batch/orchestrator** at the repo root and **single-target** at each sub-solution folder.
See [`local-build.md`](local-build.md) for the full definition of both characters, script
naming conventions, and assignment rules.

**Canonical examples:** `josyn-foundation`, `josyn-jap`

---

## Decision Rule

| Repo contains | Pattern |
|---------------|---------|
| One deliverable (library or executable), possibly with a test project | **A** |
| Multiple independently releasable deliverables sharing a domain | **B** |

When in doubt: if you would give each deliverable its own git repo in a different
organizational setup, use Pattern B.

---

## Per-Repo Assignment

| Repo | Pattern | Reason |
|------|---------|--------|
| `josyn-foundation` | **B** | Three infrastructure libraries (ResultPattern, PropertyBag, JIP) that are independently packaged and versioned. Each has its own NuGet identity. |
| `josyn-jap` | **B** | JAP shared layer groups multiple packages (Contract, Log) that are released independently but share the same protocol domain. |
| `josyn-job-host` | **A** | A single consumer-facing library (`JOSYN.JobHost`) with its test project. One deliverable, one release. |
| `josyn-backend` | **B** | `JOSYN.Jap.JAPServer` (relocated from `josyn-jap` per ADR-004) and future solutions (`SessionStarter`, `SessionStarter.Mock`) are each independent deliverables with separate deployment identities. |
| `josyn-commons` | **A** *(future)* | Utility helper library. A single growable assembly. When projects are added, Pattern A applies. |
| `josyn-sandbox` | **B** | Explicitly a sandbox for multiple independent experiments. Each experiment is a self-contained solution that evolves on its own. |

---

## Known Violations

*None. All repos are currently compliant.*
