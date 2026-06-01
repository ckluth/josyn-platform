# PROP-001 — JAPServer Relocation into josyn-backend

**Date:** 2026-06-01
**Status:** Accepted — superseded by `decisions/ADR-004-japserver-relocation.md`
**Relates to:** josyn-jap, josyn-backend

---

## Context

JAPServer currently lives in `josyn-jap` as a first-class architectural component,
symmetric with `josyn-job-host`. Its `IJosynApplicationProtocol` implementation is
stubbed (fake arguments, console output for results).

As the platform matures, JAPServer must do real work:

| Protocol member | Real requirement |
|-----------------|-----------------|
| `GetArguments()` | Read job arguments from the session-store database |
| `PutResult()` | Write job result back to the session-store |
| `GetConfigValue(string path)` | Read from the company configuration database |
| `PutExecutionReport()` | Send email notification; write to configured file share |

All real data sources (`session-store`, `company-config`) are owned by `josyn-backend`.
This creates a **sharing gap**: JAPServer needs backend resources but has no path to them.

---

## Options Considered

### Option 1 — NuGet packages (shared contracts/clients)

Introduce shared packages (e.g., `SessionStore.Client`, `CompanyConfig.Client`) that both
`josyn-backend` and `josyn-jap` reference.

**Pro:** Clean API surface; explicit contracts.  
**Con:** Requires a third shared layer (new repo or awkward placement). Is fundamentally
the same coupling problem restated — `josyn-jap` gains a compile-time dependency on
backend-owned concerns, regardless of how the packages are distributed.

### Option 2 — IPC (backend service ↔ JAPServer)

Backend exposes a singleton IPC service; JAPServer calls it at runtime.

**Pro:** Keeps processes fully independent; no code coupling.  
**Con:** Requires new infrastructure (N-client-capable transport, distinct from the
session-scoped JIP). The 1-to-N concurrency model (one backend service, many concurrent
JAPServer instances) is manageable but non-trivial to build and operate. Adds a new
runtime dependency that can fail.

### Option 3 — Direct database access

JAPServer reads/writes the session-store and config databases directly using the session GUID as key.

**Pro:** No new packages; no new IPC.  
**Con:** JAPServer must know DB schema and connection strings — a responsibility leak.
Schema coupling replaces code coupling. Any schema change requires coordinated deployment
of both backend and JAPServer.

### Option 4 (Proposed) — Relocate JAPServer into josyn-backend

Move `JOSYN.Jap.JAPServer` (the EXE source) from `josyn-jap` into `josyn-backend`.
`josyn-jap` retains only `JOSYN.Jap.Shared.Contract` and `JOSYN.Jap.Shared.Log`.
`josyn-backend` gains a dependency on those two shared packages.

**Pro:**
- JAPServer's implementation has direct access to all backend infrastructure.
- No new IPC protocol. No schema coupling. No reverse NuGet dependency.
- Runtime topology is **unchanged**: JAPServer still runs as a separate process,
  session-isolated by GUID. The process boundary is fully preserved.
- Backend's new dependency on `jap.shared` is downward and intentional.

**Con:**
- `josyn-jap`'s identity shifts: no longer a "server repo", now a "protocol contracts repo".
- The architectural symmetry between `josyn-jap` and `josyn-job-host` breaks.
- Backend gains additional NuGet dependencies (`JIP`, `PropertyBag`, `Contract`, `Log`).

---

## Why not consolidate josyn-jap into foundation or commons?

With JAPServer gone, `josyn-jap` holds only two packages. Relocating them was considered:

- **`josyn-foundation`** — would carry an evolving, protocol-specific concern
  (`IJosynApplicationProtocol`). Foundation's character is bedrock primitives — eternal
  and stable. A protocol contract that grows with the platform does not belong here.
- **`josyn-commons`** — an optional toolbox of domain-agnostic utilities.
  A JAP-specific protocol contract is neither optional nor domain-agnostic. No match.

**Conclusion:** `josyn-jap` survives as the **JAP protocol repo** — the single source of
truth for the contract shared by the two parties that speak JAP: `josyn-job-host` (client)
and `josyn-backend` (server, via the relocated JAPServer).

---

## Proposed Architecture Change

### Dependency chain (after relocation)

```
josyn-jap (contracts only)
├── JOSYN.Jap.Shared.Contract   ← IJosynApplicationProtocol, ErrorReport
└── JOSYN.Jap.Shared.Log        ← LocalLog

josyn-backend
├── JOSYN.Backend.SessionStarter
├── JOSYN.Backend.SessionStore    (future)
├── JOSYN.Backend.CompanyConfig   (future)
├── ... other backend components ...
└── JOSYN.Jap.JAPServer           ← relocated here; has full access to backend resources
    depends on: JIP, PropertyBag, ResultPattern, Contract, Log

josyn-job-host
└── JOSYN.JobHost
    depends on: JIP, PropertyBag, ResultPattern, Contract, Log  ← unchanged
```

### What does NOT change

- JAPServer.exe still spawned by backend at session start
- Session isolation by GUID unchanged
- Named pipe protocol (JIP) unchanged
- `josyn-job-host` unchanged
- `josyn-jap` shared packages unchanged

---

## Open Questions

- Does `josyn-backend` require a new subdirectory structure to accommodate the JAPServer EXE
  alongside its library packages?
- Should the JAPServer EXE remain in a separate build unit within the backend repo, or be
  integrated into the existing solution?
- No migration is needed immediately — JAPServer remains stubbed until backend components
  (`SessionStore`, `CompanyConfig`) are ready to be implemented.
