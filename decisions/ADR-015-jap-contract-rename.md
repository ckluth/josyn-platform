# ADR-015 — Rename JOSYN.Jap.Shared.Contract → JOSYN.Jap.Contract

**Date:** 2026-06-10
**Status:** Accepted

---

## Context

`josyn-jap` originally contained three packages:

| Package | Role |
|---------|------|
| `JOSYN.Jap.Shared.Contract` | JAP protocol contract — `IJosynApplicationProtocol`, `ErrorReport` |
| `JOSYN.Jap.Shared.Log` | Process-local file logger |
| `JOSYN.Jap.JAPServer` | Per-session JAP server EXE |

The `Shared.*` grouping segment was meaningful: it distinguished the neutral, multi-consumer
packages from the server-side executable. "Shared" contrasted with "not shared".

Two subsequent decisions removed that contrast:

- **ADR-004** relocated `JOSYN.Jap.JAPServer` to `josyn-backend`. The "non-shared" component
  left the repo.
- **ADR-008** retired `JOSYN.Jap.Shared.Log` and rehomed it in `josyn-commons` as
  `JOSYN.Commons.Log`. The sibling shared package left the repo.

After both decisions `josyn-jap` holds exactly one package. There is no non-shared counterpart
to contrast against. `Shared` is a historical grouping artifact that no longer groups anything.

---

## Decision

Rename the package, assembly, namespace, and directory:

```
JOSYN.Jap.Shared.Contract  →  JOSYN.Jap.Contract
```

This is a pure rename — no API, behaviour, or dependency changes.

---

## Rationale

1. **"Shared" has no referent.** With one package in the repo the grouping word is noise.
2. **Repo identity alignment.** ADR-004 reframes `josyn-jap` as "the JAP protocol contracts
   repo". The package name now matches that identity directly.
3. **Pattern compliance.** `JOSYN.Jap.Contract` follows `JOSYN.<Layer>.<Component>` cleanly,
   without an intermediate grouping segment.
4. **"Contract" is the precise signal.** It names what the package *is*. "Shared" named a
   property of how it was *used* — a weaker and now vacuous signal.

Historical ADRs (ADR-004, ADR-008) that reference `JOSYN.Jap.Shared.Contract` are left
unchanged as historical records. They describe the state at the time they were written.

---

## Consequences

- **Breaking NuGet rename.** All consumers update their `PackageReference` from
  `JOSYN.Jap.Shared.Contract` to `JOSYN.Jap.Contract`.
- **Namespace change.** All `using JOSYN.Jap.Shared.Contract;` statements updated to
  `using JOSYN.Jap.Contract;` in `josyn-job-host` and `josyn-backend`.
- **Version.** Package retains version `1.0.0-preview01` — same version, new identity in the local feed.
  The old `JOSYN.Jap.Shared.Contract` cache entry is cleared via `clean.cmd`.
- All documentation, sanity criteria, and architecture diagrams updated in the same commit.
