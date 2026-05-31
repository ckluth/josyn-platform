# Sanity — Trigger Table

Used by smart-mode sanity runs to infer which check categories need re-running
based on changed file paths in a subject repo.

---

## How it works

Changed file paths (from git log + working tree status since the last check) are matched
against the patterns below. All matching patterns contribute their categories — the result
is a union. The same file can match multiple patterns; all triggered categories are included.

An explicit `--check` or `--profile` flag **intersects** with the inferred set:
only categories that appear in **both** the inferred set and the requested set will run.
Use `--force` to bypass inference entirely and run exactly what is specified.

---

## Trigger table

| Changed path pattern | Triggered categories | Notes |
|---|---|---|
| `**/*.cs` (non-test source) | `principles` · `standards` · `docs` · `tests` · `architecture` | Source changes can affect all five dimensions |
| `**/*.Test/**/*.cs` · `**/*.Tests/**/*.cs` | `tests` | Test project files; architecture/principles not inferred |
| `**/*.csproj` · `**/*.props` · `**/*.slnx` · `**/*.sln` | `architecture` · `standards` · `docs` | Project structure, references, build config |
| `**/*.md` (in subject repo) | `docs` | Markdown documentation currency |

Patterns are applied recursively. A deletion, rename, or new file is treated the same as a modification.

---

## Identifying test files

Test projects are identified by their folder path containing `.Test` or `.Tests`:

- `JOSYN.Foundation.Test/SomeTest.cs` → test file (triggers `tests` only)
- `JOSYN.Foundation/SomeClass.cs` → non-test source (triggers all five)

When in doubt, treat the file as non-test source (more conservative).

---

## Special cases

### "Never run" baseline

If a repo's `sanity/current-state/<repo>.md` reports `**Last checked:** never`,
skip the trigger-table inference for that repo and propose a full check instead —
filtered by any `--check`/`--profile` intersection in effect.

### No changes detected

If no file changes are found since the last check, the repo is current.
Skip it, note it in the confirmation gate proposal, and still update its
`**Last checked:**` field with a `(verified current at <now>)` suffix.

### Partial-failure

If inference or the check run itself fails for a repo, mark it as `⚠️ stale` in
`sanity/overview.md`. Do not overwrite its `sanity/current-state/<repo>.md` with
incomplete findings — leave the previous state intact.
