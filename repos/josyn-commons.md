# josyn-commons

**Role:** Domain-agnostic platform utilities — open for growth.
Consumed by any repo as NuGet packages. Never referenced by `josyn-foundation`.

**Location:** `C:\Users\chris\OneDrive\DevGit\josyn-commons`
**Version:** `1.0.0-preview01`

---

## Architectural position

`josyn-commons` is the **domain-agnostic platform utilities** layer of the JOSYN platform.

```
JOSYN.Commons.*  (no deps, or ResultPattern only)
      ▲  ▲  ▲
      │  │  │
   (any josyn repo may reference it — foundation never does)
```

Unlike `josyn-foundation`, which is a **stable-forever** infrastructure layer, `josyn-commons`
is **open for growth**: new helpers are added over time as common patterns emerge across repos.
All additions must be backward-compatible — existing consumers are never broken.

See [decisions/ADR-003-josyn-commons.md](../decisions/ADR-003-josyn-commons.md) for the
full rationale behind this layer.

---

## Rules

### What belongs here

A helper belongs in `josyn-commons` if and only if:

1. It is reusable across **≥ 2** josyn repos.
2. It carries **no JOSYN-domain knowledge** — no sessions, no jobs, no IPC, no scheduling.
3. It could theoretically live in **any C# project**, not just JOSYN.

Helpers that do not meet all three criteria stay in the consuming repo.

### Dependency constraint

`josyn-commons` packages may only reference:

- **Nothing** (preferred), or
- **`JOSYN.Foundation.ResultPattern`** (when a helper needs to express failure as a value)

`JOSYN.Foundation.PropertyBag`, `JOSYN.Foundation.JIP`, and any package from `josyn-jap`,
`josyn-job-host`, or `josyn-backend` are **forbidden** dependencies. Violating this rule
would pull `josyn-commons` out of its bottom-of-DAG position.

---

## Packages

| Package | Sub-folder | Status |
|---------|-----------|--------|
| `JOSYN.Commons.Log` | `josyn-commons-log/` | Existing |

Each package follows the naming pattern:

```
josyn-commons-<topic>/   →  JOSYN.Commons.<Topic>
```

---

## Build & Package

Each sub-package follows the same layout as `josyn-foundation`:

```
.local-build/build.cmd [Release|Debug]
.local-build/test.cmd
.local-build/pack.cmd              ← outputs to ../../local-packages/
```

License: MIT | Company: HAEVG AG | Target: net10.0

---

## Sanity Notes

### Current state

- `JOSYN.Commons.Log` — process-local file logger; migrated from `JOSYN.Jap.Shared.Log` per ADR-008.

### Admission criteria (for any new package)
Before any new package is added, verify it meets **all three** criteria:
1. Reusable across ≥ 2 josyn repos.
2. Carries no JOSYN domain knowledge — no sessions, no jobs, no IPC, no scheduling.
3. Could live in any C# project, not just JOSYN.

A package that fails any criterion must stay in the consuming repo instead.

### Dependency constraints
- Permitted references: **nothing** (preferred), or `JOSYN.Foundation.ResultPattern` only.
- `JOSYN.Foundation.PropertyBag`, `JOSYN.Foundation.JIP`, and any package from `josyn-jap`, `josyn-job-host`, or `josyn-backend` are **forbidden**.
- `josyn-foundation` must **never** reference `josyn-commons` — this would invert the dependency hierarchy.

