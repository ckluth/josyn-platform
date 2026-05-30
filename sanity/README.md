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

## Command surface

```
run sanity-check [--check <category>] [--repo <repo-name>]
```

Both flags are **optional** and **repeatable**. Omitting a flag means *all* of that axis.

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
