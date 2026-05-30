# Sanity Check — Method and Manual

The sanity check is a structured, agent-driven review of the JOSYN platform codebase.
It is defined and owned by `josyn-platform`. All other repos are subjects of the check.

---

## What it checks

Five orthogonal categories, each with its own criteria file:

| Category | File | What it covers |
|----------|------|----------------|
| `docs` | `docs.md` | XML documentation comments and markdown documentation currency |
| `tests` | `tests.md` | Unit test existence, naming, structure, coverage, and passing state |
| `principles` | `principles.md` | Functional-first C# coding rules — Result pattern, static-first, immutability, no-throw |
| `architecture` | `architecture.md` | NuGet dependency chain integrity and forbidden cross-repo references |
| `standards` | `standards.md` | Project file conventions, naming rules, and directory structure |

Per-repo specifics (known exceptions, baseline expectations, repo-specific constraints) are in
`../repos/<repo-name>.md` under the `## Sanity Notes` section of each file.

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
```

`--profile`, `--check`, and `--repo` are all **optional** and **repeatable**.
`--profile` and `--check` can be combined — the union of categories is used.
Omitting both `--profile` and `--check` means *all* categories.

### Examples

```
run sanity-check
```
Full check — all five categories across all five repos. Use this for a periodic platform-wide review.

```
run sanity-check --repo josyn-foundation
```
All categories, one repo. Use after changes to `josyn-foundation`.

```
run sanity-check --check principles
```
One category, all repos. Use to verify a platform-wide rule has been applied consistently.

```
run sanity-check --check docs --repo josyn-jap
```
Focused check — one category, one repo. Use during a grooming pass on a specific area.

```
run sanity-check --check docs --check tests --repo josyn-foundation --repo josyn-jap
```
Multiple categories and multiple repos in one run.

### Valid repo names

`josyn-foundation` · `josyn-jap` · `josyn-job-host` · `josyn-backend` · `josyn-commons`

`josyn-platform` is never a target — it is the owner of the check, not a subject.

### Valid category names

`docs` · `tests` · `principles` · `architecture` · `standards`

---

## Safety contract — READ-ONLY

**A sanity check never modifies anything. This is an absolute rule, even with `/autopilot` active.**

The agent executing a sanity check is permitted to:
- ✅ Read any file in any repo
- ✅ Run `dotnet test` to verify test state (produces no file changes in the repos)

The agent is **never** permitted to:
- ❌ Create, edit, or delete any file
- ❌ Run any command with side effects beyond test execution
- ❌ Fix violations — report only

---

## Running unattended (nap mode)

To run a full sanity check without any permission interruptions:

```
/autopilot
run sanity-check --profile full
```

`/autopilot` auto-approves all agent actions. The safety contract above guarantees no writes occur.
To exit autopilot after the check: `/autopilot` again (it toggles).

---

## How an agent executes a sanity check

1. **Read `../AGENTS.md`** — orientation, repo paths, and the full execution protocol.
2. **Load criteria** — for each requested category, read the corresponding `<category>.md` file in this folder.
   Some categories require additional context files (listed in AGENTS.md execution steps).
3. **Load per-repo notes** — for each requested repo, read `../repos/<repo-name>.md`,
   specifically the `## Sanity Notes` section, to understand known exceptions and repo-specific expectations.
4. **Inspect each repo** — apply the loaded criteria to the actual code in the repo at its relative path.
5. **Report findings** — grouped by repo, then by category:
   - List every **violation** explicitly with file path and reason.
   - Confirm every **clean area** explicitly — silence is not a pass.
   - Note any area that **could not be verified** (e.g. repo not found, build broken) and why.

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

## Keeping criteria current

The criteria files in this folder are the **single source of truth** for what "correct" means.
When a platform rule changes:

1. Update the relevant criteria file here.
2. Update `../architecture/coding-standards.md` or `../architecture/naming-conventions.md` if the rule originates there.
3. The change takes effect for all future sanity runs immediately — no other configuration needed.
