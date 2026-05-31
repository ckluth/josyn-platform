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

---

## 5. Knowledge Map

No content is duplicated here. Read the canonical source when a topic requires detail.

| Topic | Canonical source |
|-------|-----------------|
| Architecture, runtime flow, component map | `architecture/overview.md` |
| Naming conventions, namespace rules, directory structure | `architecture/naming-conventions.md` |
| Coding standards, principles, Result pattern reference | `architecture/coding-standards.md` |
| Architectural decision records | `decisions/` |
| Per-repo summaries (packages, assemblies, current state) | `repos/` |

---

## 6. Sanity Check Protocol

### Command surface

```
run sanity-check [--profile <name>] [--check <category>] [--repo <repo-name>]
run sanity-check --deep [--profile <name>] [--check <category>] [--repo <repo-name>]
run sanity-check --force [--profile <name>] [--check <category>] [--repo <repo-name>]
```

`--profile`, `--check`, and `--repo` are all optional and repeatable.
`--profile` and `--check` can be combined — the union of categories is used as the filter.
Omitting `--repo` means all repos.

| Flag | Behaviour |
|------|-----------|
| *(none)* | **Smart default** — trigger-table inference; proposes a targeted run |
| `--deep` | **Deep smart** — full diff analysis, semantic inference; warns about extra cost |
| `--force` | **Forced** — no inference; runs exactly what is specified (or everything if no filters) |

`--deep` and `--force` cannot be combined — reject with a clear error message.

### Profiles

| Profile | Expands to | Cost |
|---------|-----------|------|
| `--profile quick` | `architecture` + `standards` | ⚡ fast |
| `--profile code` | `principles` + `docs` | 🐢 slow |
| `--profile full` | all 5 categories | 🐢 slowest |

### Repos in scope

The five subject repos (`josyn-platform` itself is never a sanity-check target):

`josyn-foundation` · `josyn-jap` · `josyn-job-host` · `josyn-backend` · `josyn-commons`

### Check categories

| Category | Criteria file | Scope |
|----------|--------------|-------|
| `docs` | `sanity/criteria/docs.md` | XML comments, markdown documentation currency |
| `tests` | `sanity/criteria/tests.md` | Unit test existence, coverage, passing state |
| `principles` | `sanity/criteria/principles.md` | Result pattern, static-first, immutability, no-throw |
| `architecture` | `sanity/criteria/architecture.md` | Dependency chain integrity, forbidden references |
| `standards` | `sanity/criteria/standards.md` | Naming, project file conventions, directory structure |

### Smart inference (default mode)

**Before** loading criteria or opening subject repos, the agent runs a preflight inference phase:

1. For each repo in scope: read `sanity/current-state/<repo>.md` — extract the `**Last checked:**`
   timestamp. If the value is `never`, skip inference for this repo and propose a full check
   (filtered by any `--check`/`--profile` intersection in effect).
2. Collect changed file paths in the subject repo since that timestamp:
   - `git log --since=<timestamp> --name-only --pretty=format:` for committed changes
   - `git status --short` for staged, unstaged, untracked, deleted, and renamed files
3. Apply the trigger table in `sanity/triggers.md` to the changed paths — derive the categories
   that need re-checking.
4. If `--check` or `--profile` flags were given, **intersect** the derived categories with the
   requested ones — only run categories that appear in both sets.
5. If no changes are detected in a repo → mark it as current in the proposal; skip its checks.
6. If all repos are current → report that everything is up to date; no run is needed unless
   `--force` is used.

### Deep inference (`--deep`)

Identical to smart inference steps 1–2, then:

3. Read the actual diffs (`git diff` and `git diff --staged`) for each changed file.
4. Reason semantically about the nature of each change — new public API, deleted tests,
   added `throw`, changed project reference, etc. — to derive categories.
5. Apply `--check`/`--profile` intersection as in smart step 4 above.
6. At the confirmation gate, note: *"Deep analysis read N diffs before this proposal."*

### Force execution (`--force`)

No inference phase. Run exactly the repos and categories specified.
If neither `--check` nor `--profile` is given, run all five categories across all repos.

### Confirmation gate

**After** the inference phase (or immediately for `--force`), the agent **must**:

1. **For `--force` runs:** perform a quick change scan (git log + git status, no category inference)
   for each target repo. Include the result per repo in the gate message:
   - Repos with changes since last check: list the count of changed files.
   - Repos with **no** changes since last check: flag with ⚠️.
   - Repos with `Last checked: never`: no flag (no baseline to compare against).
2. Explicitly state what will run — repos, categories, cost estimate, and which repos (if any)
   were skipped as current (smart/deep mode) or flagged as unchanged (force mode).
3. Ask: *"Really run this now?"* and wait for explicit confirmation.
4. If any 🐢 category is involved across multiple repos, also suggest:
   *"This is a long runner — consider `/autopilot` so it runs unattended."*

**Exception:** If autopilot is already active, skip steps 1–4 and proceed directly.

### Safety contract

A sanity check writes only to this repo, never to any subject repo:

- `sanity/current-state/<repo>.md` — one file per repo that was checked, overwritten with findings
- `sanity/overview.md` — regenerated from all five state files at the end of every run

`dotnet test` is the only command with side effects beyond reading and writing these two targets.

### Execution steps

1. Read this file (done).
2. Run the **inference phase** (smart or deep) — produce the proposed run scope.
3. Present the **confirmation gate** — wait for approval before proceeding.
4. For each requested **category**: read `sanity/criteria/<category>.md`.
   - Before `principles`: also read `architecture/coding-standards.md`.
   - Before `architecture`: also read `architecture/overview.md`.
   - Before `standards` or `docs`: also read `architecture/naming-conventions.md`.
5. For each requested **repo**: open the repo at its relative path and apply all loaded criteria.
   Per-repo detail is in `repos/<repo-name>.md`.
6. Collect all findings.
7. Overwrite `sanity/current-state/<repo>.md` for each checked repo. Set `**Last checked:**` to
   the current UTC timestamp. Confirm clean areas — silence is not a pass.
8. If a repo was skipped as current, update its `**Last checked:**` field to
   `<previous-timestamp> (verified current at <now>)` — do not alter its findings.
9. Regenerate `sanity/overview.md` from all five current-state files.

### Fixing violations

```
fix violations [--repo <repo-name>] [--check <category>]
```

Source of truth is always `sanity/current-state/<repo>.md` for the target repo.
Never fix from `sanity/overview.md` (summary only) or from memory.
Fix one repo/category pair at a time, verify with a targeted re-run, commit before moving to the next.

**Interaction rule:** Fixing is always step-by-step and interactive. For each violation, propose the exact change and wait for explicit confirmation before executing. Never batch fixes without asking.

Full strategy: `sanity/README.md` → *Fixing violations* section.

⚠️ Fixing is a write operation — do not use `/autopilot`. Stop and ask before refactoring `principles` violations.
