# josyn-commons

**Role:** Domain-agnostic platform utilities — open for growth.
Consumed by any repo as NuGet packages. Never referenced by `josyn-foundation`.

**Location:** `C:\DevGit\josyn-commons`
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
| `JOSYN.Commons.Helpers` | `josyn-commons-helpers/` | Existing — `Turnstile` only |
| `JOSYN.Commons.IdentityHelpers` | `josyn-commons-identity-helpers/` | Existing — `WindowsCredential`, `ImpersonatedProcess` |
| `JOSYN.Commons.Schedule` | `josyn-commons-schedule/` | Existing — INI-based time-schedule parser, serializer, validator (ADR-026) |

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
- `JOSYN.Commons.Helpers` — `Turnstile`; introduced by ADR-019. `WindowsCredential` and `ImpersonatedProcess` were extracted into `JOSYN.Commons.IdentityHelpers`.
- `JOSYN.Commons.IdentityHelpers` — `WindowsCredential` (validated UPN value type) and `ImpersonatedProcess` (Windows process launch under a domain account); extracted from `JOSYN.Commons.Helpers` after ADR-021/ADR-022.
- `JOSYN.Commons.Schedule` — INI-based time-schedule parser (`ScheduleParser`), serializer (`ScheduleSerializer`), and semantic validator (`ScheduleValidator`); implements the schedule definition language from ADR-026. Consumed by `TimeScheduler.exe`. The package is generic (no session/job/IPC knowledge) and carries only the ability to read and write the schedule notation.

### Admission criteria (for any new package)
Before any new package is added, verify it meets **all three** criteria:
1. Reusable across ≥ 2 josyn repos.
2. Carries no JOSYN domain knowledge — no sessions, no jobs, no IPC, no platform-scheduling
   logic. A library that parses or formats a generic notation (such as a time-schedule
   format) is not "scheduling" in this sense; it has no awareness of how jobs are
   orchestrated or executed.
3. Could live in any C# project, not just JOSYN.

A package that fails any criterion must stay in the consuming repo instead.

### Dependency constraints
- Permitted references: **nothing** (preferred), or `JOSYN.Foundation.ResultPattern` only.
- `JOSYN.Foundation.PropertyBag`, `JOSYN.Foundation.JIP`, and any package from `josyn-jap`, `josyn-job-host`, or `josyn-backend` are **forbidden**.
- `josyn-foundation` must **never** reference `josyn-commons` — this would invert the dependency hierarchy.

