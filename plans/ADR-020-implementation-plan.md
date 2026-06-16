# ADR-020 Implementation Plan — Company Adapter Model (Out-of-Process)

**Created:** 2026-06-16  
**Last updated:** 2026-06-16 (Phase 8 complete — ADR-020 fully implemented)  
**ADR:** `josyn-platform/decisions/ADR-020-company-adapter-model.md`  
**ADR-017B-03:** `josyn-platform/decisions/ADR-017B-03-credential-provider.md`

---

## Progress snapshot

| Phase | Title | Status |
|-------|-------|--------|
| 0 | BootstrapConfig cleanup | ✅ Done |
| 1 | BootstrapConfig `[Adapters]` section | ✅ Done |
| 2 | JAPServer adapter process management | ✅ Done |
| 3 | ADR-017B-03 decision | ✅ Done |
| 4 | IdentityAdapter contract package | ✅ Done |
| 5 | JAPServer IdentityAdapter call site | ✅ Done |
| 6 | Contoso IdentityAdapter stub EXE | ✅ Done |
| 7 | HAEVG IdentityAdapter EXE | ⏸ Deferred |
| 8 | Documentation updates | ✅ Done |

---

## Context

ADR-020 is accepted. The old ALC adapter artifacts (AdapterLoadContext, ContosoConfigSource,
IConfigSource, SqlConfigSource, SessionStarter) are deleted. ConfigStore was refactored to read
SQL directly. Phases 0–5 are fully implemented and build clean.

---

## ✅ Phase 0 — BootstrapConfig cleanup (complete)

---

Removed `ConfigSourceType` from `IBootstrapConfig`, `FileBootstrapConfig`, and `josyn.bootstrap.ini`.
Re-packed `JOSYN.Backend.BootstrapConfig`. JAPServer rebuilt clean.

---

## ✅ Phase 1 — BootstrapConfig `[Adapters]` section (complete)

Added `IReadOnlyDictionary<string, string> Adapters { get; }` to `IBootstrapConfig`.
`FileBootstrapConfig` switches from `DeserializeSingleSection` to full multi-section `Deserialize`.
Reads sectionless root keys into `_root`; reads `[Adapters]` section keys into `_adapters`.
Updated `josyn.bootstrap.ini` with commented-out `[Adapters]` example block.
Re-packed `JOSYN.Backend.BootstrapConfig`. JAPServer rebuilt clean.

**Note:** The plan originally described a flat `Adapters.IdentityAdapter = ...` prefix approach.
During implementation, the user changed the design to a proper `[Adapters]` INI section — which
scales better and required the multi-section parser switch.

---

## ✅ Phase 2 — JAPServer adapter process management (complete)

New files created:
- `AdapterProcess.cs` — wraps spawned `Process` + `ClientPipes` + session `Guid`; `IDisposable`
- `AdapterManager.cs` — dictionary keyed by concern name; `GetPipes(name)`; `IDisposable`
- `Host.Adapters.cs` — `SpawnAdapters(IBootstrapConfig)`: resolves EXE from `Adapters/` folder,
  spawns with `JOSYN-ADAPTER <guid>`, connects JIP; hard failure on missing/unresponsive adapter;
  defines `IdentityAdapterConcern = "IdentityAdapter"` constant

Updated:
- `Host.Entrypoint.cs` — `using var adapterManager = SpawnAdapters(...)` before session start
- `JAPServer.cs` — added `AdapterManager adapterManager` constructor param
  (had `#pragma warning disable CS9113` comment until Phase 5 wired the call site)

Build clean, 0 warnings.

---

## ✅ Phase 3 — ADR-017B-03 decision (complete)

ADR-017B-03 rewritten from placeholder to Accepted.

Decisions made:
- Concern name: `IdentityAdapter`
- Contract: `GetPassword(string username) → Result<string>` — single JIP method
- Package: `JOSYN.Backend.IdentityAdapter.Contract`
- No platform stub; Contoso owns the stub
- HAEVG implementation deferred
- Domain not included in Phase 5 scope

---

## ✅ Phase 4 — IdentityAdapter contract package (complete)

Renamed `josyn-backend-adapter-contracts/JOSYN.Backend.AdapterContracts` →
`josyn-backend-identity-adapter-contract/JOSYN.Backend.IdentityAdapter.Contract`.

Full Pattern B structure created:
- `JOSYN.Backend.IdentityAdapter.Contract.slnx`
- `nuget.config`
- `JOSYN.Backend.IdentityAdapter.Contract.csproj`
- `IIdentityAdapter.cs` — single method: `Task<Result<string>> GetPassword(string username)`
- `.local-build/build.cmd`, `pack.cmd`, `clean.cmd`, `all.cmd`

Built and packed to local feed successfully.

---

## ✅ Phase 5 — JAPServer IdentityAdapter call site (complete)

Added `JOSYN.Backend.IdentityAdapter.Contract` NuGet reference to JAPServer `.csproj`.

Changes in `Host.Prepare.cs`:
- `PrepareContext` gains `string? TechnicalUserPassword` field
- New `ResolveCredentials` async phase: gets `IdentityAdapter` pipes from `AdapterManager`,
  calls `JipClient.SendAsync("GetPassword", TechnicalUserName)`, stores result in context
- Uses `Host.IdentityAdapterConcern` constant — no magic strings
- `TODO (ADR-017B-03)` comment in `LaunchJobAndStorePid` marks where impersonated spawn will go
  (impersonation itself is **deferred** — password is resolved but not yet used for spawn identity)

Removed `#pragma warning disable CS9113` from `JAPServer.cs` — `adapterManager` is now used.

Build clean, 0 warnings.

**Important deferred item:** Impersonation (running `job.exe` under the technical user's identity)
is explicitly NOT implemented. The `TODO` comment marks the exact location. This is a separate
platform decision that requires OS-level impersonation design (Windows `CreateProcessWithLogonW`,
etc.) and is out of scope for ADR-020.

---

## ✅ Phase 6 — Contoso IdentityAdapter stub EXE (complete)

**Repo:** `josyn-contoso`

New folder: `josyn-contoso/contoso-identity-adapter/` (Pattern B structure)

Files created:
- `Contoso.IdentityAdapter.slnx`
- `nuget.config` → `..\..\local-packages` (= `C:\DevGit\local-packages`)
- `.local-build/build.cmd`, `clean.cmd`
- `Contoso.IdentityAdapter/Contoso.IdentityAdapter.csproj` — EXE, net10.0
- `Contoso.IdentityAdapter/Program.cs` — parses `JOSYN-ADAPTER <guid>`, registers handler, runs `PipesServer.RunAsync`
- `Contoso.IdentityAdapter/PasswordStore.cs` — loads flat INI, implements `IIdentityAdapter`
- `Contoso.IdentityAdapter/contoso.credentials.ini` — sample credentials (copied to output)

Dependencies: `JOSYN.Foundation.JIP` + `JOSYN.Backend.IdentityAdapter.Contract`.

**Note:** `IIdentityAdapter.GetPassword(string)→Task<Result<string>>` is not in `JipDispatcher.RegisterAll`'s
supported signatures (it handles zero-param or `string`-in/`Result`-out, not `string`-in/`Result<string>`-out).
Handler is registered manually via `dispatcher.Register(name, Func<string?, Task<Result<string?>>>)`.

Also packed `JOSYN.Backend.IdentityAdapter.Contract` to `C:\DevGit\local-packages` (Phase 4's pack.cmd had
not been run against the shared feed).

Root `.local-build` updated:
- `build.cmd` — `contoso-adapter` → `contoso-identity-adapter`
- `clean.cmd` — `contoso-adapter` → `contoso-identity-adapter`
- `pack.cmd` — removed dead `contoso-adapter` call (EXE produces no NuGet package)

Build clean, 0 warnings.

**Bootstrap config required for Contoso/dev deployments:**
```ini
[Adapters]
IdentityAdapter = Contoso.IdentityAdapter.exe
```

---

## ⏸ Phase 7 — HAEVG IdentityAdapter EXE (deferred)

Real company adapter reading credentials from company infrastructure (Credential Manager, vault, etc.).
Repo TBD. Blocked on HAEVG-specific credential storage design.

---

## ⬜ Phase 8 — Documentation updates (complete)

- `repos/josyn-backend/overview.md`:
  - Added ADR-017B-03 (renamed title) and ADR-020 to decisions table
  - Added `josyn-backend-identity-adapter-contract/` to repo file tree
  - Updated JAPServer purpose/dependencies to include `JOSYN.Backend.IdentityAdapter.Contract`
  - Replaced outdated Host.cs pseudocode with current partial-class structure:
    `Host.Entrypoint`, `Host.Adapters`, `Host.Prepare`, `AdapterManager`, `AdapterProcess`
  - Replaced JIP dispatcher wiring section with new "Adapter protocol (ADR-020)" section
  - Added `IdentityAdapter.Contract` to planned components table (Done)
  - Updated JAPServer dependencies XML block
  - Updated runtime spawn relationships (adapters added)
  - Updated sanity notes permitted references
- `architecture/overview.md`:
  - Updated component map: added `JOSYN.Backend.IdentityAdapter.Contract`, split
    JAPServer internals, added `Adapters/` deployment folder
  - Updated spawn relationships: `JOSYN-START`, adapter spawn, job spawn
- `ROADMAP.md`:
  - Updated "What's done": ADR-020, IdentityAdapter.Contract, Contoso stub
  - Removed `JOSYN.Backend.AdapterContracts` from undocumented components list
    (renamed/replaced by `IdentityAdapter.Contract`, now documented)

---

## Next session starting point

**Start here:** Phase 8 — Documentation updates (see above for full spec).

Phase 6 is complete and builds clean. Phase 7 (HAEVG) remains deferred indefinitely.

**Key files to load at session start:**
- This file
- `josyn-platform/decisions/ADR-020-company-adapter-model.md`
- `josyn-platform/decisions/ADR-017B-03-credential-provider.md`
- `josyn-contoso/contoso-identity-adapter/Contoso.IdentityAdapter/Program.cs`
- `josyn-contoso/contoso-identity-adapter/Contoso.IdentityAdapter/PasswordStore.cs`
