# ADR-017 — Introduce `JOSYN.Backend.Contracts` as the Single Backend Contracts Package

**Date:** 2026-06-12
**Status:** Accepted

---

## Context

The JOSYN backend layer currently has its contracts distributed across multiple locations:

| Artefact | Current home | Problem |
|---|---|---|
| `SessionStartRequest` | `JOSYN.Backend.SessionLauncherContract` | One record per package — does not scale |
| `IConfigSource` | `JOSYN.Backend.AdapterContracts` | One interface per package — does not scale |
| `IJobSessionRecord` | `JOSYN.Backend.SessionStore` | A domain contract buried inside an implementation package |
| `ExecutionStatus` (new, ADR-016) | No home yet | Needs a package; belongs to no existing package correctly |

The immediate trigger is ADR-016: `ExecutionStatus` is an enum that represents a core domain
concept. It is written by `SessionStarter`, transitioned by `JAPServer`, and read by any
consumer of session records. Placing it in `JOSYN.Backend.SessionStore` would couple all
status-reading consumers to the storage implementation package — the wrong dependency direction.

The pattern for resolving this already exists one layer down: `JOSYN.Jap.Contract` is the
single package for all JAP protocol contracts. There is no `JOSYN.Jap.ProtocolContract`,
`JOSYN.Jap.ErrorContract`, and `JOSYN.Jap.EnvironmentContract` — one package holds them all.
The backend layer should follow the same pattern.

---

## Decision

Introduce **`JOSYN.Backend.Contracts`** as the single package for all backend-layer contracts,
domain vocabulary, and shared data types.

### Contents at introduction

| Artefact | Source |
|---|---|
| `IJobSessionRecord` | Moved from `JOSYN.Backend.SessionStore` |
| `JobSessionRecord` | Moved from `JOSYN.Backend.SessionStore` |
| `SessionStartRequest` | Moved from `JOSYN.Backend.SessionLauncherContract` |
| `IConfigSource` | Moved from `JOSYN.Backend.AdapterContracts` |
| `ExecutionStatus` | New — enum, ADR-016 |
| `ExecutionStatusParser` | New — `Result<ExecutionStatus> Parse(string)` + `string Serialize(ExecutionStatus)` |

### Retirement

`JOSYN.Backend.SessionLauncherContract` and `JOSYN.Backend.AdapterContracts` are retired.
Their packages are removed; their solution sub-folders are deleted.

### `JOSYN.Backend.SessionStore` after the move

`SessionStore` loses its public contract types (`IJobSessionRecord`, `JobSessionRecord`) and
gains a dependency on `JOSYN.Backend.Contracts`. It becomes a pure implementation package:
EF Core internals, `ISessionStore`, `SessionStore`. `ISessionStore` remains in `SessionStore`
because it is the store's own service interface — not a cross-cutting domain contract.

### Namespace

All types in `JOSYN.Backend.Contracts` use the namespace `JOSYN.Backend.Contracts`.
Consumers update `using` statements accordingly.

---

## Rationale

1. **Mirrors the established `JOSYN.Jap.Contract` pattern.** One contracts package per
   layer is the platform-wide convention. Fragmented per-concept packages are an
   anti-pattern already corrected once (ADR-015).

2. **Contracts belong upstream of implementations.** `IJobSessionRecord` is consumed by
   `JAPServer` and `SessionStarter` — neither should depend on the storage implementation
   package for a type definition.

3. **`ExecutionStatus` has no correct existing home.** It is consumed by every layer that
   touches a session. A single, stable contracts package is the only correct destination.

4. **Do it now.** The platform has not reached production. The consumer count is small and
   fully known. The cost of consolidation is low today; it compounds every sprint we defer.

---

## Affected consumers

| Consumer | Change |
|---|---|
| `JOSYN.Jap.JAPServer` | `AdapterContracts` + `SessionLauncherContract` → `Contracts`; `SessionStore` reference retained (for `ISessionStore`) |
| `JOSYN.Backend.SessionStarter` | `SessionStore` reference retained (for `ISessionStore`); gains `Contracts` reference for `ExecutionStatus` |
| `JOSYN.Backend.SessionLauncher` | `SessionLauncherContract` → `Contracts` |
| `JOSYN.Backend.ConfigStore` | `AdapterContracts` → `Contracts` |
| `Contoso.Josyn.Adapter` | `AdapterContracts` → `Contracts` |

---

## Consequences

- `JOSYN.Backend.Contracts` becomes the first reference any new backend-layer contract
  or domain type lands in — no new single-concept contracts package is ever created for
  this layer.
- `JOSYN.Backend.SessionLauncherContract` and `JOSYN.Backend.AdapterContracts` are
  deleted in the same session as this ADR is accepted.
- `josyn-contoso` is an outside consumer of `AdapterContracts`; its `PackageReference`
  is updated in the same session.
- All historical ADRs referencing `JOSYN.Backend.AdapterContracts` or
  `JOSYN.Backend.SessionLauncherContract` are left unchanged as historical records.
