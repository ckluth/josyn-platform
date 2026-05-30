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

### Confirmation gate

Before executing any `run sanity-check` command, the agent **must**:

1. Explicitly paraphrase the intended run — which repos, which categories, and whether the cost
   table indicates a 🐢 long run.
2. Ask: *"Really run this now?"* and wait for explicit confirmation.
3. If the expected runtime includes any 🐢 category across multiple repos, also suggest:
   *"This is a long runner — consider `/autopilot` so it runs unattended."*

**Exception:** If autopilot is already active, skip steps 1–3 and proceed directly. Autopilot is
the deliberate opt-in to unattended execution; confirmation would contradict it.

### Safety contract — READ-ONLY

A sanity check makes exactly one write: `sanity/last-result.md` in this repo is overwritten with the
findings at the end of every run. No other files are created, edited, or deleted — not in this repo,
not in any subject repo. `dotnet test` is the only command with side effects beyond reading and writing
that one result file.

### Execution steps

1. Read this file (done).
2. For each requested **category**: read `sanity/<category>.md` to load the criteria.
   - Before `principles`: also read `architecture/coding-standards.md`.
   - Before `architecture`: also read `architecture/overview.md`.
   - Before `standards` or `docs`: also read `architecture/naming-conventions.md`.
3. For each requested **repo**: open the repo at its relative path and apply all loaded criteria.
   Per-repo detail (assemblies, packages, current state) is in `repos/<repo-name>.md`.
4. Collect all findings.
5. Overwrite `sanity/last-result.md` with the structured report (see `sanity/README.md` for format).
   Confirm clean areas — silence is not a pass.

### Fixing violations

```
fix violations [--repo <repo-name>] [--check <category>]
```

Source of truth is always `sanity/last-result.md`. Fix one repo/category pair at a time,
verify with a targeted re-run, commit before moving to the next.
Full strategy: `sanity/README.md` → *Fixing violations* section.

⚠️ Fixing is a write operation — do not use `/autopilot`. Stop and ask before refactoring `principles` violations.
