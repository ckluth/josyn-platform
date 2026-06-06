# ADR-008 — LocalLog Relocation to josyn-commons

**Date:** 2026-06-05
**Status:** Proposed

---

## Context

`LocalLog` (from `JOSYN.Jap.Shared.Log`) is the platform's process-local file logger.
It is a static utility: writes timestamped entries to a structured log directory, supports
per-causer subdirectories, swallows I/O errors silently, and optionally mirrors output to
the console in debug builds.

It is already consumed by three distinct repos:

| Consumer | Repo |
|----------|------|
| `JOSYN.Jap.JAPServer` | `josyn-backend` |
| `JOSYN.JobHost` | `josyn-job-host` |
| `JOSYN.Jap.Shared.Log.Test` | `josyn-jap` |

A fourth consumer is imminent: `JOSYN.Backend.ErrorHandler` (ADR-006) requires `LocalLog`
as its last-resort fallback when SQL storage is unavailable.

`LocalLog` currently lives in `JOSYN.Jap.Shared.Log` — a package whose identity is the
JAP protocol layer. This is a misplacement: `LocalLog` carries no JAP protocol knowledge,
no session context, no IPC concern. It is a general-purpose file logger that happens to
have been written alongside the JAP shared packages.

Leaving it there creates two concrete problems:

1. **Wrong dependency signal.** Any repo that needs `LocalLog` must take a NuGet dependency
   on a package named `JOSYN.Jap.Shared.Log`. This implies a JAP protocol relationship
   where none exists.

2. **DRY violation risk.** `JOSYN.Backend.ErrorHandler` cannot use a JAP-named package
   for its fallback without the wrong dependency signal. The alternative — a self-contained
   fallback implementation — creates two diverging implementations of the same behaviour.

---

## Decision

### 1. LocalLog moves to josyn-commons as `JOSYN.Commons.Log`

`LocalLog` and its companion interface `ILocalLog` are extracted from `JOSYN.Jap.Shared.Log`
and re-homed in a new `josyn-commons` package: `JOSYN.Commons.Log`.

The public API, behaviour, and dependency (ResultPattern only) are unchanged.

```
josyn-commons/
└── josyn-commons-log/
    ├── nuget.config
    ├── JOSYN.Commons.Log.slnx
    ├── .local-build/
    └── JOSYN.Commons.Log/
        ├── Contracts/
        │   └── ILocalLog.cs
        └── LocalLog.cs
```

### 2. josyn-commons identity is reframed

`josyn-commons` is not an "optional" satellite — it is the **domain-agnostic platform utilities**
layer. The distinction from `josyn-foundation` is not importance but scope: foundation packages
are intrinsic to the JOSYN communication model (result propagation, serialization, IPC transport);
commons packages are general-purpose helpers reusable across the platform.

The admission criteria are unchanged:

1. Reusable across ≥ 2 josyn repos.
2. Carries no JOSYN domain knowledge — no sessions, no jobs, no IPC, no scheduling.
3. Could theoretically live in any C# project, not just JOSYN.

The dependency constraint is unchanged: commons packages may only reference nothing or
`JOSYN.Foundation.ResultPattern`. Foundation never references commons.

### 3. JOSYN.Jap.Shared.Log is retired

`JOSYN.Jap.Shared.Log` contained only `LocalLog` and `ILocalLog`. Once these are moved,
the package has no remaining content. It is retired: the project is removed from
`josyn-jap`, the NuGet package is withdrawn from the local feed.

All consumers are updated in the same migration:

| Consumer | Change |
|----------|--------|
| `JOSYN.Jap.JAPServer` | `PackageReference` updated from `JOSYN.Jap.Shared.Log` → `JOSYN.Commons.Log` |
| `JOSYN.JobHost` | same |
| `JOSYN.Backend.ErrorHandler` | references `JOSYN.Commons.Log` from first version |

### 4. josyn-jap documentation is updated

`repos/josyn-jap.md` is updated to remove the `JOSYN.Jap.Shared.Log` section and reflect
the retirement. The repo description is updated: "JAP protocol contracts — shared contract
package (`JOSYN.Jap.Shared.Contract`) only."

---

## Decision challenge — objections and rebuttals

**Attacker:** `josyn-commons` is described as an optional utility satellite. Making `LocalLog`
live there gives it a load-bearing role in the error handler fallback path. That weakens the
"optional" character of commons — once a critical component depends on it, it is no longer
truly optional.

**Defender:** The "optional" label was always a perception, not a hard rule. The only hard rule
is that `josyn-foundation` never references `josyn-commons`. Everything else may reference it.
`josyn-commons` is optional in the sense that no project is *forced* to use it — not in the
sense that it is unimportant. Reframing it as "domain-agnostic platform utilities" removes the
false impression without changing any architectural constraint.

---

**Attacker:** The simpler fix is to keep `LocalLog` in `josyn-jap` and just have
`JOSYN.Backend.ErrorHandler` take a dependency on `JOSYN.Jap.Shared.Log`. That dependency
direction is already established — `JAPServer` does it. No migration, no new package.

**Defender:** `JAPServer` takes that dependency because it is a JAP protocol server — the
dependency is semantically correct for it. `JOSYN.Backend.ErrorHandler` is a pure backend
component with no JAP protocol concern. Forcing it to depend on a JAP-named package to get
a file logger sends the wrong signal to every future reader of the dependency graph. The
naming matters: a developer seeing `JOSYN.Jap.Shared.Log` as a dependency of the error
handler will correctly ask "why?", and the honest answer is "misplacement". Moving the package
fixes the root cause rather than papering over it.

---

**Attacker:** The alternative is a self-contained fallback write inside `ErrorHandler` — a
private five-line method that writes a timestamped line to a file. No migration, no new
package, no dependency at all. The existing `FileSystemErrorHandler` stub already shows this
pattern.

**Defender:** That creates two independent implementations of process-local file logging in
the same platform — one in `JOSYN.Jap.Shared.Log` (or its successor) and one embedded in
`ErrorHandler`. They will diverge: different formats, different path strategies, different
silent-failure handling. A new developer will not know which to trust or extend. A documented
DRY violation is still a DRY violation — the documentation does not eliminate the maintenance
cost, it only makes it visible. The one-time migration cost of moving `LocalLog` is smaller
than the permanent maintenance burden of two diverging implementations.

---

## Consequences

- All consumers of `LocalLog` use the same implementation with no naming confusion
- `JOSYN.Jap.Shared.Log` is retired cleanly — no dormant package left in the feed
- `josyn-commons` receives its first real package, validating the layer's existence
- The dependency signal is correct: consumers of `LocalLog` depend on `JOSYN.Commons.Log`,
  not on a JAP protocol package
- `JOSYN.Backend.ErrorHandler` can reference `JOSYN.Commons.Log` without architectural compromise
- The "domain-agnostic platform utilities" reframing of commons removes the false impression
  that commons packages are unimportant or skippable
