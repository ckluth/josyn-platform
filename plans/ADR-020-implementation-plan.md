# ADR-020 Implementation Plan — Company Adapter Model (Out-of-Process)

**Created:** 2026-06-16  
**Last updated:** 2026-06-16 (end-of-session checkpoint)  
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
| 6 | Contoso IdentityAdapter stub EXE | ⬜ Not started |
| 7 | HAEVG IdentityAdapter EXE | ⏸ Deferred |
| 8 | Documentation updates | ⬜ Not started |

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

## ⬜ Phase 6 — Contoso IdentityAdapter stub EXE

**Repo:** `josyn-contoso`

**Goal:** Contoso provides a stub `IdentityAdapter.exe` that satisfies the contract for
standalone/dev deployments.

**Agreed design:**
- Console EXE: `Contoso.IdentityAdapter.exe JOSYN-ADAPTER <guid>`
- Reads passwords from a local INI file (e.g. `contoso.credentials.ini`) — flat key-value,
  username → password
- References `JOSYN.Backend.IdentityAdapter.Contract` and the JIP server-side infrastructure
- Pattern B structure: own `.slnx`, own `nuget.config`, own `.local-build/`

**Work:**
- [ ] Create `josyn-contoso/contoso-identity-adapter/` subfolder with Pattern B structure
- [ ] Console EXE project `Contoso.IdentityAdapter`
- [ ] `IIdentityAdapter` JIP dispatcher: parses `username` from `data`, looks up in INI, returns password
- [ ] `contoso.credentials.ini` — example/template credential file (gitignored in real deployments)
- [ ] `.local-build/build.cmd`, `pack.cmd` (if distributable), `clean.cmd`
- [ ] Add `contoso.credentials.ini` template to `josyn-contoso` root (or next to EXE in output)

**Entry point pattern** (consistent with JAPServer adapter CLI convention):
```csharp
// args[0] = "JOSYN-ADAPTER", args[1] = <session-guid>
// Start JipServer with that guid, register IIdentityAdapter dispatcher, block until pipe closes
```

---

## ⏸ Phase 7 — HAEVG IdentityAdapter EXE (deferred)

Real company adapter reading credentials from company infrastructure (Credential Manager, vault, etc.).
Repo TBD. Blocked on HAEVG-specific credential storage design.

---

## ⬜ Phase 8 — Documentation updates

- [ ] `repos/josyn-backend/overview.md` — add `IdentityAdapter.Contract`; add `AdapterManager`,
  `AdapterProcess`, `Host.Adapters`; remove stale ALC/ConfigSource entries
- [ ] `architecture/overview.md` — update component map to show adapter EXEs under JAPServer
- [ ] `ROADMAP.md` — mark ADR-020 phases complete

---

## Next session starting point

**Start here:** Phase 6 — Contoso IdentityAdapter stub EXE (see above for full spec).

After Phase 6 is complete and tested end-to-end, do Phase 8 (docs).
Phase 7 (HAEVG) remains deferred indefinitely.

**Key files to load at session start:**
- This file
- `josyn-platform/decisions/ADR-020-company-adapter-model.md`
- `josyn-platform/decisions/ADR-017B-03-credential-provider.md`
- `josyn-backend/josyn-backend-jap-server/JOSYN.Jap.JAPServer/Host.Prepare.cs`
- `josyn-backend/josyn-backend-jap-server/JOSYN.Jap.JAPServer/Host.Adapters.cs`
- `josyn-backend/josyn-backend-identity-adapter-contract/JOSYN.Backend.IdentityAdapter.Contract/IIdentityAdapter.cs`
