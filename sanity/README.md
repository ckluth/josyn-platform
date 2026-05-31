# Sanity Check — Method and Manual

The sanity check is a structured, agent-driven review of the JOSYN platform codebase.
It is defined and owned by `josyn-platform`. All other repos are subjects of the check.

---

## What it checks

Five orthogonal categories, each with its own criteria file:

| Category | File | What it covers |
|----------|------|----------------|
| `docs` | `criteria/docs.md` | XML documentation comments and markdown documentation currency |
| `tests` | `criteria/tests.md` | Unit test existence, naming, structure, coverage, and passing state |
| `principles` | `criteria/principles.md` | Functional-first C# coding rules — Result pattern, static-first, immutability, no-throw |
| `architecture` | `criteria/architecture.md` | NuGet dependency chain integrity and forbidden cross-repo references |
| `standards` | `criteria/standards.md` | Project file conventions, naming rules, and directory structure |

Per-repo specifics (known exceptions, baseline expectations, repo-specific constraints) are in
`../repos/<repo-name>.md` under the `## Sanity Notes` section of each file.

The meta-quality of the criteria themselves is tracked in [`criteria-review.md`](criteria-review.md) —
a living document to be revisited periodically by the platform maintainer.

---

## Cost overview

Not all checks are equal in effort. Use this table to choose the right scope:

| Category | 1 repo | n repos | all repos |
|----------|--------|---------|-----------|
| `architecture` | ⚡ quick | ⚡ quick | ⏱ takes a while |
| `standards` | ⚡ quick | ⏱ takes a while | ⏱ takes a while |
| `docs` | ⏱ takes a while | ⏱ takes a while | 🐢 long runner |
| `tests` | ⏱ takes a while | 🐢 long runner | 🐢 long runner |
| `principles` | 🐢 long runner | 🐢 long runner | 🐢 long runner |

---

## Profiles

Named shortcuts for common category combinations:

| Profile | Expands to | When to use |
|---------|-----------|-------------|
| `--profile quick` | `--check architecture --check standards` | After any structural change — rename, move, new project |
| `--profile code` | `--check principles --check docs` | After a coding session — did the code stay clean? |
| `--profile full` | all 5 categories | Scheduled review, before a release, full platform sanity |

Profiles can be combined with `--repo` to scope them to a subset of repos:
```
run sanity-check --profile quick --repo josyn-backend
run sanity-check --profile code --repo josyn-foundation --repo josyn-jap
```

---

## Command surface

```
run sanity-check [--profile <name>] [--check <category>] [--repo <repo-name>]
run sanity-check --deep [--profile <name>] [--check <category>] [--repo <repo-name>]
run sanity-check --force [--profile <name>] [--check <category>] [--repo <repo-name>]
```

`--profile`, `--check`, and `--repo` are all **optional** and **repeatable**.
`--profile` and `--check` can be combined — the union of categories is used as the filter.
Omitting `--repo` means all repos.

| Flag | Behaviour |
|------|-----------|
| *(none)* | **Smart default** — trigger-table inference; proposes only what has changed |
| `--deep` | **Deep smart** — full diff analysis, semantic inference; warns about extra cost |
| `--force` | **Forced** — no inference; runs exactly what is specified |

`--deep` and `--force` cannot be combined — rejected with a clear error message.

### Examples

```
run sanity-check
```
Smart default — infers what changed since the last check across all repos. Proposes only the
relevant repos and categories. If nothing has changed, reports that everything is up to date.

```
run sanity-check --repo josyn-foundation
```
Smart, scoped to one repo. Infers which categories need re-running based on its changes.

```
run sanity-check --check principles
```
Smart, `principles` only. Considers all repos — but only proposes a `principles` run for those
where source code changes have been detected (intersection of inferred + requested).

```
run sanity-check --deep --repo josyn-jap
```
Deep analysis of `josyn-jap`. Reads actual diffs to determine which categories to run.

```
run sanity-check --force
```
Full forced run — all five categories across all five repos. No inference, no filtering.

```
run sanity-check --force --check docs --repo josyn-foundation
```
Forced `docs` check on `josyn-foundation`. Skips inference, runs exactly this scope.

```
run sanity-check --check docs --check tests --repo josyn-foundation --repo josyn-jap
```
Smart, two categories, two repos. Only runs where changes match either `docs` or `tests`.

### Valid repo names

`josyn-foundation` · `josyn-jap` · `josyn-job-host` · `josyn-backend` · `josyn-commons`

`josyn-platform` is never a target — it is the owner of the check, not a subject.

### Valid category names

`docs` · `tests` · `principles` · `architecture` · `standards`

---

## Smart inference

Before loading any criteria or opening subject repos, the agent runs a **preflight inference phase**
to determine what actually needs to be checked.

### Smart default (no `--force`, no `--deep`)

1. For each repo in scope: read `sanity/current-state/<repo>.md` — extract the `**Last checked:**`
   timestamp.
   - If `never`: skip inference for this repo; propose a full check (filtered by any
     `--check`/`--profile` intersection in effect).
2. Collect changed file paths in the subject repo since that timestamp:
   - `git log --since=<timestamp> --name-only --pretty=format:` for committed changes
   - `git status --short` for staged, unstaged, untracked, deleted, and renamed files
3. Apply the trigger table in `sanity/triggers.md` to derive which categories need re-checking.
4. If `--check` or `--profile` flags were given, **intersect** the derived categories with the
   requested ones — only run categories that appear in both sets.
5. If no changes are detected in a repo → skip it; note it as current in the proposal.
6. If all repos are current → report that everything is up to date; no run is needed.

### Deep inference (`--deep`)

Identical to steps 1–2 above, then:

3. Read the actual diffs (`git diff` and `git diff --staged`) for each changed file.
4. Reason semantically about the nature of each change — new public API, deleted tests,
   added `throw`, changed project reference, etc. — to derive categories.
5. Apply `--check`/`--profile` intersection as above.
6. At the confirmation gate, note: *"Deep analysis read N diffs before this proposal."*

### Force execution (`--force`)

No inference phase. Omitting `--check`/`--profile`/`--repo` means run all five categories
across all five repos.

---

## State files

Every run writes to two locations — both within `josyn-platform`, never in subject repos:

- `sanity/current-state/<repo>.md` — one file per checked repo, overwritten with findings
- `sanity/overview.md` — aggregated dashboard, regenerated from all five state files at end of run

`sanity/last-result.md` is retired. It contains a redirect notice only.

### Per-repo state file format

```markdown
# Sanity State — <repo-name>

**Last checked:** 2026-05-31T12:00:00Z

## Summary

| Category | Status |
|----------|--------|
| docs | ✅ |
| tests | ✅ |
| principles | ❌ 2 violations |
| architecture | ✅ |
| standards | ✅ |

---

## docs
  ✅ All XML comments present and correct.

## principles
  ❌ <file path> — <reason>
```

### Overview file format

```markdown
# Sanity Check — Platform Overview

**Generated:** 2026-05-31T12:00:00Z

| Repo | docs | tests | principles | architecture | standards | Last checked |
|------|------|-------|-----------|-------------|-----------|-------------|
| josyn-foundation | ✅ | ✅ | ❌ | ✅ | ✅ | 2026-05-31T12:00:00Z |
```

### Skipped-as-current repos

When a repo has no changes since its last check, do not re-run checks for it.
Update its `**Last checked:**` field to:

```
2026-05-20T10:00:00Z (verified current at 2026-05-31T12:00:00Z)
```

Do not alter its findings.

---

## Safety contract

A sanity check writes only to this repo, never to any subject repo:

- `sanity/current-state/<repo>.md` for each checked repo
- `sanity/overview.md` — regenerated at the end of every run

No other files are created, edited, or deleted anywhere.

---

## Confirmation gate

**After** the inference phase (or immediately for `--force`), the agent must:

1. **State what will run** — repos, categories, cost estimate, and which repos (if any) were
   skipped as current.
2. **Ask for confirmation:** *"Really run this now?"* — wait for an explicit yes before proceeding.
3. If any 🐢 category is involved across more than one repo, also **suggest autopilot:**
   *"This is a long runner — consider `/autopilot` so it runs unattended without further interruptions."*
4. If `--deep` was given, note upfront: *"Deep analysis will read diffs before proposing a run."*

**Exception:** if autopilot is already active, skip confirmation entirely and proceed.

---

## Running unattended (nap mode)

To run a full sanity check without any permission interruptions:

```
/autopilot
run sanity-check --profile full
```

`/autopilot` auto-approves all agent actions. The safety contract above guarantees no writes
occur outside the two state files in this repo.

---

## How an agent executes a sanity check

1. **Read `../AGENTS.md`** — orientation, repo paths, and the full execution protocol.
2. **Run the inference phase** — smart (trigger-table) or deep (diff analysis) before loading
   criteria or opening subject repos. Produce the proposed run scope.
3. **Present the confirmation gate** — state what will run and wait for approval.
4. **Load criteria** — for each requested category, read `criteria/<category>.md`.
   Some categories require additional context files (listed in AGENTS.md execution steps).
5. **Load per-repo notes** — for each requested repo, read `../repos/<repo-name>.md`,
   specifically the `## Sanity Notes` section.
6. **Inspect each repo** — apply the loaded criteria to the actual code in the repo at its
   relative path.
7. **Write findings** — overwrite `sanity/current-state/<repo>.md` for each checked repo.
   Set `**Last checked:**` to the current UTC timestamp. Grouped by category:
   - List every **violation** explicitly with file path and reason.
   - Confirm every **clean area** explicitly — silence is not a pass.
   - Note any area that **could not be verified** (e.g. repo not found, build broken) and why.
8. **Update skipped repos** — for repos with no changes, update their `**Last checked:**` field
   with a `(verified current at <now>)` suffix. Do not alter findings.
9. **Regenerate `sanity/overview.md`** — aggregate from all five state files.

---

## Reading the report

A compliant finding looks like:

```
josyn-foundation / principles
  ✅ All public types are static or record — no unjustified instance classes found.
  ✅ No throw statements outside lowest-layer catch boundaries.
  ✅ All failure paths return Result or Result<T>.
```

A violation looks like:

```
josyn-jap / docs
  ❌ JOSYN.Jap.Shared.Contract/IJosynApplicationProtocol.cs
     GetRawArguments() — <summary> missing on interface method.
  ❌ JOSYN.Jap.Shared.Log/LocalLog.cs
     WriteError(string causer, string message) — implementation duplicates
     interface doc text instead of using <inheritdoc/>.
```

An unverifiable area looks like:

```
josyn-backend / tests
  ⚠️ No test project found. Known gap — PoC stub phase. See repos/josyn-backend.md Sanity Notes.
```

---

## Fixing violations

### Command

```
fix violations [--repo <repo-name>] [--check <category>]
```

Both flags are optional. Omitting means all violations across all repos.
Combine with `--repo` and `--check` to scope the fix session to a single area.

### Source of truth

Always fix from `sanity/current-state/<repo>.md` for the target repo.
Never fix from `sanity/overview.md` (summary only) or from memory.
If the state file is stale, run a fresh sanity check first.

### Workflow — one repo/category pair at a time

1. **Load violations** — read `sanity/last-result.md`, filter by requested scope.
2. **Load context** — read the relevant `sanity/criteria/<category>.md` criteria file and `repos/<repo>.md` Sanity Notes.
3. **Check for known exceptions** — if a violation is documented as expected state in Sanity Notes, skip it. Do not fix expected state.
4. **Fix** — apply the minimal change that resolves the violation in the target repo.
5. **Verify** — re-run `run sanity-check --check <category> --repo <repo>` immediately after fixing.
   The fix is done only when the check reports ✅ for that area.
6. **Commit** — one commit per verified repo/category pair. Do not batch fixes across categories.
7. Repeat for the next violation.

### Priority order

Fix in this order to avoid compounding issues:

| Priority | Category | Reason |
|----------|----------|--------|
| 1st | `architecture` | Broken dependency edges can prevent builds |
| 2nd | `standards` | Missing scaffolding and wrong project config affects all other checks |
| 3rd | `docs` | Mechanical — low risk, high clarity gain |
| 4th | `tests` | Requires judgment — discuss missing coverage before writing |
| 5th | `principles` | Highest reasoning required — never bulk-fix without human review |

### Safety

Fixing is a **write operation**. Do not use `/autopilot` for a fix session.
Every change must be reviewed by the human before committing.

`principles` violations in particular must be discussed before fixing — a refactor
that looks mechanical (e.g. "make this class static") may have non-obvious consequences.
Stop and ask when in doubt.

---

The criteria files in `criteria/` are the **single source of truth** for what "correct" means.
When a platform rule changes:

1. Update the relevant criteria file here.
2. Update `../architecture/coding-standards.md` or `../architecture/naming-conventions.md` if the rule originates there.
3. The change takes effect for all future sanity runs immediately — no other configuration needed.

---

## Scope boundary — josyn-platform is not a subject

The sanity check covers the five subject repos only. `josyn-platform` itself — its own markdown files,
criteria files, and documentation — is never evaluated by a sanity run.

This is a deliberate blind spot. The platform repo is the *owner* of the check, not a subject of it.
Consistency and quality of the platform's own files is a human responsibility, maintained through
normal code review when the repo is changed.

