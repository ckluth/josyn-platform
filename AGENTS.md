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
| `josyn-jap` | Per-session JAP server — JAPServer EXE, shared contracts, logging | `../josyn-jap` |
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

## 3. Agent Behavior

- Stay sceptical — do not be a pleaser.
- Always explain your plan before running anything.
- When in doubt about intent or scope: stop and ask.
- You are not the final decision-maker — the human always reviews.

---

## 4. Knowledge Map

No content is duplicated here. Read the canonical source when a topic requires detail.

| Topic | Canonical source |
|-------|-----------------|
| Architecture, runtime flow, component map | `architecture/overview.md` |
| Naming conventions, namespace rules, directory structure | `architecture/naming-conventions.md` |
| Coding standards, principles, Result pattern reference | `architecture/coding-standards.md` |
| Architectural decision records | `decisions/` |
| Per-repo summaries (packages, assemblies, current state) | `repos/` |

---

## 5. Sanity Check Protocol

### Command surface

```
run sanity-check [--profile <name>] [--check <category>] [--repo <repo-name>]
```

All flags are optional and repeatable. `--profile` and `--check` can be combined (union of categories).
Omitting both `--profile` and `--check` means all categories. Omitting `--repo` means all repos.

### Profiles

| Profile | Expands to | Cost |
|---------|-----------|------|
| `--profile quick` | `architecture` + `standards` | ⚡ fast |
| `--profile code` | `principles` + `docs` | 🐢 slow |
| `--profile full` | all 5 categories | 🐢 slowest |

```
run sanity-check                                          # all checks, all repos
run sanity-check --repo josyn-foundation                  # all checks, one repo
run sanity-check --check docs                             # one category, all repos
run sanity-check --check docs --repo josyn-foundation      # one category, one repo
run sanity-check --check docs --check tests               # two categories, all repos
```

### Repos in scope

The five subject repos (`josyn-platform` itself is never a sanity-check target):

`josyn-foundation` · `josyn-jap` · `josyn-job-host` · `josyn-backend` · `josyn-commons`

### Check categories

| Category | Criteria file | Scope |
|----------|--------------|-------|
| `docs` | `sanity/docs.md` | XML comments, markdown documentation currency |
| `tests` | `sanity/tests.md` | Unit test existence, coverage, passing state |
| `principles` | `sanity/principles.md` | Result pattern, static-first, immutability, no-throw |
| `architecture` | `sanity/architecture.md` | Dependency chain integrity, forbidden references |
| `standards` | `sanity/standards.md` | Naming, project file conventions, directory structure |

### Safety contract — READ-ONLY

A sanity check is inspect-and-report. There is nothing to write — no fixes, no edits, no file creation. `dotnet test` is the only command with side effects (test execution produces no file changes in the repos).

### Execution steps

1. Read this file (done).
2. For each requested **category**: read `sanity/<category>.md` to load the criteria.
   - Before `principles`: also read `architecture/coding-standards.md`.
   - Before `architecture`: also read `architecture/overview.md`.
   - Before `standards` or `docs`: also read `architecture/naming-conventions.md`.
3. For each requested **repo**: open the repo at its relative path and apply all loaded criteria.
   Per-repo detail (assemblies, packages, current state) is in `repos/<repo-name>.md`.
4. Report findings **per repo, then per category**. List every violation explicitly.
   Confirm clean areas — silence is not a pass.
