# ADR-005B-01 — Backend Building Block Model

**Date:** 2026-06-02
**Status:** Accepted

---

## Context

`josyn-backend` is the outermost layer of the JOSYN platform. It is growing beyond a single
stub. Before real components are added, the architectural model governing what kinds of
building blocks live here — and how they are structured — must be explicit and recorded.

The backend will accumulate two fundamentally different types of components:

- **Reusable logic** that other backend components share (candidates: `SessionStore`,
  `SessionStarter`, `GlobalConfig`)
- **Independent processes** that are spawned at runtime (`JAPServer` already exists;
  `TriggerAgent`, `Service` are planned)

Without a clear model, the boundary between these two categories risks becoming blurry,
and the NuGet/solution/dependency discipline that the platform relies on could erode.

---

## Decision

### 1. Two categories of building blocks

All backend building blocks belong to exactly one of two categories:

**Category A — Shared libraries (NuGet artifacts)**

- Packaged as NuGet and consumed by other backend components via `PackageReference`
- Define stable contracts (interfaces) that decouple consumers from implementations
- May have multiple implementations (production + mock)
- Examples: `JOSYN.Backend.SessionStore`, `JOSYN.Backend.GlobalConfig`

**Category B — Executables**

- Standalone OS processes; never packaged as NuGet
- Spawned at runtime by other components; not referenced via `PackageReference`
- Consume shared libraries (Category A) as NuGet dependencies
- Examples: `JOSYN.Jap.JAPServer`, `JOSYN.Backend.TriggerAgent`, `JOSYN.Backend.Service`

### 2. JAPServer's special role

`JOSYN.Jap.JAPServer` is a Category B executable with a unique position: it is the only
executable in this repo that is also a JAP protocol implementation. It:

- Lives in `josyn-backend` (not `josyn-jap`) because it requires direct access to backend
  resources (`SessionStore`, `CompanyConfig`) — per ADR-007B-01
- Is spawned by `SessionStarter`; not by any other executable in this repo
- Is the single point where the JAP protocol meets backend state

No other executable in this repo shares this dual role.

### 3. Contract/implementation split for shared libraries

Every Category A component exposes its functionality through a stable interface. The
canonical layout within its project:

```
JOSYN.Backend.<Component>/
├── I<Component>.cs       ← the contract
└── <Component>.cs        ← production implementation
```

Mock implementations live in a dedicated companion library:

```
JOSYN.Backend.<Component>.Mock/
└── Mock<Component>.cs    ← mock implementation (for development and demo)
```

The mock is a separate NuGet artifact so production assemblies never carry test or demo
dependencies.

**`SessionStore` is the first real component and the reference implementation of this pattern:**

- `JOSYN.Backend.SessionStore` — contract (`ISessionStore`) + production implementation
  (EF Core, reading from the SQL session-store table of the legacy job system)
- `JOSYN.Backend.SessionStore.Mock` — mock implementation for development and demo use

### 4. One solution per building block

Each building block gets its own `.slnx` solution file. Cross-solution dependencies always
go through NuGet — never via project references across solution boundaries.
`.local-build/build.cmd` builds all solutions in the repo.

---

## Consequences

- The boundary between shared state (NuGet library) and process behaviour (executable) is
  always unambiguous when a new component is introduced
- Mock implementations are always available without polluting production packages
- `ISessionStore` (and future `IGlobalConfig`, etc.) are stable contracts that `JAPServer`
  and `SessionStarter` can depend on without knowing which implementation is active at runtime
- The solution-per-building-block rule keeps solutions small and independently buildable
- `josyn-backend` can grow without architectural drift: every new component is classified
  on arrival as Category A or Category B
