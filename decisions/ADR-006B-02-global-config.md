# ADR-006B-02 — GlobalConfig

**Date:** 2026-06-03
**Status:** Accepted

---

## Context

`JOSYN.Backend.GlobalConfig` is the second Category A component in `josyn-backend`,
as established by ADR-005B-01. Multiple backend components — starting with `SessionStore` and
the planned `SessionStarter` — require shared runtime configuration: at minimum a database
connection string and the path to the `JAPServer.exe` binary.

Without a stable interface, each component would either hardcode these values internally or
invent its own ad-hoc mechanism to receive them. The interface must be established now,
even as a placeholder, so that all consumers are wired correctly from day one and no
component ever owns a configuration concern that belongs elsewhere.

---

## Decisions

### 1. A dedicated package with a stable interface

`IGlobalConfig` is the single configuration contract for the backend. It lives in its own
NuGet package (`JOSYN.Backend.GlobalConfig`) so that it can be versioned and consumed
independently. Any backend component that needs configuration takes an `IGlobalConfig`
parameter — never a raw string, never a static field from another component.

### 2. `HardcodedGlobalConfig` as the PoC implementation

The real configuration mechanism (file-based, registry, or company config system) is a
future concern. At PoC stage, `HardcodedGlobalConfig` supplies compile-time constants:
a developer-machine connection string and the known local release output path for
`JAPServer.exe`.

This approach is the same class of known limitation as the fake session key in
`JOSYN.Jap.JAPServer` — an intentional PoC shortcut, explicitly documented, not a
production pattern.

### 3. No NuGet dependencies

`IGlobalConfig` exposes only `string` properties. No third-party library is needed.
Adding `JOSYN.Foundation.ResultPattern` would be artificial coupling with no benefit.
This package has zero NuGet dependencies.

### 4. `IGlobalConfig` is a regular instance interface

This is a genuine runtime polymorphism seam: future implementations (file-based, registry,
company config system) will replace `HardcodedGlobalConfig` by supplying a different
`IGlobalConfig` instance at composition root. `static abstract` interfaces are not
appropriate here — they are for static type contracts, not for swappable runtime
implementations.

### 5. `JapServerExePath` is a developer-machine path in the PoC implementation

The hardcoded path `C:\Temp\VS.OUT\JOSYN\JOSYN.Jap.JAPServer\bin\Release\...` is
Release-only and machine-specific. This is intentional and documented — the path will
be supplied by a real config source when the platform leaves the PoC phase.

---

## Consequences

- All backend consumers of connection strings or runtime paths depend on `IGlobalConfig`,
  never on hardcoded literals scattered across components
- Replacing the PoC `HardcodedGlobalConfig` with a real implementation requires only a
  change at the composition root — all consumers remain untouched
- `JOSYN.Backend.GlobalConfig` has no transitive dependencies, keeping downstream
  dependency graphs clean
