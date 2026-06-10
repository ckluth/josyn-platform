# josyn-backend

**Role:** Scheduler and session-orchestration layer — the integration bridge between the
legacy job backend and the JOSYN platform. Also the home of `JOSYN.Jap.JAPServer`
(relocated from `josyn-jap` per ADR-004).

**Location:** `C:\DevGit\josyn-backend`
**Version:** `1.0.0-preview01` (stub)

---

## Decisions

Backend-specific architectural decisions are recorded in [`decisions/`](decisions/).

| ADR | Title |
|-----|-------|
| [ADR-001](decisions/ADR-001-backend-building-block-model.md) | Backend Building Block Model |
| [ADR-002](decisions/ADR-002-session-store.md) | SessionStore |
| [ADR-003](decisions/ADR-003-global-config.md) | GlobalConfig |
| [ADR-005](decisions/ADR-005-job-registry.md) | JobRegistry |
| [ADR-006](decisions/ADR-006-error-handler.md) | ErrorHandler |

---

## Architectural position

`josyn-backend` is the **outermost layer** of the JOSYN platform. It owns:

- **Scheduling** — detecting when a job should run (polling, triggers, workflow activation)
- **Session lifecycle** — creating session entries in the session store, assigning GUIDs
- **Process spawning** — launching `JAPServer.exe` with the session GUID; JAPServer in turn spawns `job.exe`
- **Result collection** — reading session completion status from the session store
- **JAPServer** — `JOSYN.Jap.JAPServer` lives in this repo (relocated per ADR-004) because it needs direct access to backend resources (`SessionStore`, `CompanyConfig`)

`josyn-jap` (shared contracts and logger packages) and `josyn-job-host` (job executable runtime)
are unaware of `josyn-backend`. The session GUID is the only runtime coupling: `josyn-backend`
passes it to `JAPServer` at spawn time; `JAPServer` forwards it to `job.exe` when it spawns it.

---

## Migration narrative — from old to new

### The old backend (logical components)

| Component | Responsibility |
|-----------|----------------|
| `JobSystem.Service` | Windows service hosting a WCF server; receives "start job" requests |
| `JobSystem.TriggerAgent` | Windows service; polls once per minute for scheduled executions |
| `JobSystem.Scheduling` | Proprietary time-scheduling mechanism |
| `JobSystem.WorkflowAdapter` | Activates a job as a step within a proprietary workflow |
| `JobSystem.Database` | SQL Server with `JobSessions` table; every session is persisted here |
| `JobSystem.JobRepository` | File-system tree; all job assemblies are deployed here |
| `JobSystem.SessionStarter` | The rendezvous: creates a session row, then `Process.Start(JobHost.exe, sessionUID)` |
| `JobSystem.JobHost` | Loads the job DLL via reflection; one isolated process per session |

### The flaw — JobHost

`JobHost.exe` was the weak point. It loaded job assemblies into its own process via reflection,
required careful assembly-resolve gymnastics, could never use modern EF Core (version conflicts),
and had to be recompiled for every .NET version. The result: a wall preventing any real evolution.

### The cure — "SessionStarter spawns JAPServer" eliminates the shared-process trap

`SessionStarter` is already the clean seam between the backend and the execution engine.
Changing what it spawns is all that is needed for a phased migration.

The real architectural cure is **process-space separation via the JIP trick**: instead of loading
job code into a shared host process (the old `JobHost`), each job now runs as a fully independent
OS process. The two processes — `JAPServer.exe` and `job.exe` — communicate exclusively through
a pair of GUID-named pipes (`JOSYN.Foundation.JIP`). No shared memory, no shared assembly load
context. Each process targets any .NET version it needs. Session isolation is structural, not
enforced by code conventions.

| | Old | New |
|---|-----|-----|
| What backend spawns | `JobHost.exe <sessionUID>` | `JAPServer.exe JOSYN-IPC <guid>` |
| What JAPServer spawns | *(n/a — JobHost loaded job via reflection)* | `job.exe JOSYN-IPC <guid>` |
| Job loading | Reflection into JobHost process space | job.exe is an independent process |
| .NET versioning | One JobHost per .NET version | Each job.exe targets its own .NET version freely |
| Backend coupling | JobHost knew the session-store schema | JAPServer owns the protocol; job.exe knows nothing of the backend |

Old and new can coexist: jobs migrated to the JOSYN model are started via the new
`SessionStarter`; legacy jobs continue to be started via the old `SessionStarter` in parallel.

---

## Current state

`JOSYN.Jap.JAPServer` is a working EXE, relocated from `josyn-jap` per ADR-004.
`JOSYN.Backend.SessionStore` is the first real Category A NuGet package, ported from the
sandbox prototype per ADR-002. (The sandbox is now called `josyn-playground`.)
`JOSYN.Backend.BootstrapConfig` is the second Category A NuGet package; provides `IBootstrapConfig`
and `FileBootstrapConfig` (reads `josyn.bootstrap.ini`), per ADR-003.
`JOSYN.Backend.SessionStarter` is the third Category A NuGet package; provides `ISessionStarter`
which persists the session and spawns `JAPServer.exe`, per ADR-002 Phase 3.
`JOSYN.Jap.JAPServer` has been fully wired: `GetRawArguments` reads from `SessionStore`,
`PutRawResult` writes back to `SessionStore` — fake methods removed.
`JOSYN.Backend.ErrorHandler` is a Category A NuGet package; provides `IErrorHandler`,
`IErrorRecord`, and `SqlErrorHandler` persisting to `josyn.ErrorStore`, per ADR-006.
`JOSYN.Jap.JAPServer` now calls `IErrorHandler.Handle()` from `PutError` and all terminal
error paths, with `jobName` and `sessionGuid` enrichment as defined in ADR-006.

```
josyn-backend/
├── .local-build/
│   ├── build.cmd                           ← builds ALL solutions in the repo
│   └── pack.cmd
├── db/                                     ← JOSYN Storage Realm scripts
│   ├── bootstrap-local-dev.sql             ← creates DB, schema, dev login (dev only)
│   └── migrations/
│       ├── V001__session_store.sql         ← josyn.SessionStore
│       ├── V002__job_registry.sql          ← josyn.JobRegistry + FK_SessionStore_JobRegistry
│       └── V003__error_store.sql           ← josyn.ErrorStore
├── local-packages/                         ← local NuGet feed
├── josyn-backend-session-store/            ← Pattern B sub-folder
│   ├── nuget.config
│   ├── JOSYN.Backend.SessionStore.slnx
│   ├── .local-build/                       ← solution-local build + pack scripts
│   └── JOSYN.Backend.SessionStore/
├── josyn-backend-bootstrap-config/         ← Pattern B sub-folder
│   ├── nuget.config
│   ├── JOSYN.Backend.BootstrapConfig.slnx
│   ├── .local-build/                       ← solution-local build + pack scripts
│   └── JOSYN.Backend.BootstrapConfig/
├── josyn-backend-session-starter/          ← Pattern B sub-folder
│   ├── nuget.config
│   ├── JOSYN.Backend.SessionStarter.slnx
│   ├── .local-build/                       ← solution-local build + pack scripts
│   └── JOSYN.Backend.SessionStarter/
├── josyn-backend-job-registry/             ← Pattern B sub-folder
│   ├── nuget.config
│   ├── JOSYN.Backend.JobRegistry.slnx
│   ├── .local-build/                       ← solution-local build + pack scripts
│   └── JOSYN.Backend.JobRegistry/
├── josyn-backend-error-handler/            ← Pattern B sub-folder
│   ├── nuget.config
│   ├── JOSYN.Backend.ErrorHandler.slnx
│   ├── .local-build/                       ← solution-local build + pack scripts
│   └── JOSYN.Backend.ErrorHandler/
├── josyn-backend-jap-server/              ← Pattern B sub-folder
│   ├── nuget.config
│   ├── JOSYN.Jap.JAPServer.slnx
│   ├── .local-build/                       ← solution-local build + launch scripts
│   └── JOSYN.Jap.JAPServer/
```

## JOSYN.Jap.JAPServer

**Purpose:** Per-session JAP protocol server executable. Listens on named pipes (JIP), receives
JAP requests from job executables, dispatches to the `IJosynApplicationProtocol` implementation.
Relocated from `josyn-jap` per ADR-004 — it needs backend resources (`SessionStore`,
`CompanyConfig`) that are owned by this repo.

**Dependencies:** `JOSYN.Foundation.JIP`, `JOSYN.Foundation.PropertyBag`,
`JOSYN.Foundation.ResultPattern`, `JOSYN.Jap.Contract`, `JOSYN.Commons.Log`,
`JOSYN.Backend.SessionStore`, `JOSYN.Backend.ErrorHandler`

**Type:** `net10.0` Console EXE

### CLI contract

```
JOSYN.Jap.JAPServer.exe JOSYN-IPC <sessionGUID>
```

Exit codes:
- `0` — success
- `1` — fatal error (missing key, IPC failure, unhandled exception)

### Structure

```csharp
// Program.cs — thin entry point
static async Task<int> Main(string[] args) => await Host.Run(args);

// Host.cs — lifecycle and dispatch
static class Host
{
    static Task<int> Run(string[] args)
    {
        // 1. Parse session key from args
        // 2. Construct HardcodedGlobalConfig + SessionStore
        // 3. Create JAPServer(sessionStore, sessionKey)
        // 4. Register all IJosynApplicationProtocol methods with JipDispatcher
        // 5. Start PipesServer — blocking until ESC or fatal error
    }
}

// JAPServer.cs — IJosynApplicationProtocol implementation
sealed class JAPServer(ISessionStore sessionStore, Guid sessionGuid) : IJosynApplicationProtocol
{
    Task<Result<string>> GetRawArguments();        // reads Arguments from SessionStore
    Task<Result> PutRawResult(string result);      // writes Result back to SessionStore
    Task<Result> PutError(string serializedError); // deserializes ErrorReport → LocalLog
}
```

### JIP dispatcher wiring

```csharp
_jipDispatcher.RegisterAll<IJosynApplicationProtocol>(jAPServer);
```

`RegisterAll` auto-discovers all methods on `IJosynApplicationProtocol` via reflection and maps them to `what` strings matching method names.

### Known limitations

- Demo session key: `dea5611d-d740-437f-ad93-7a5dc5ae4299` in `launchSettings.json`

---

## Planned components

When migrated, `josyn-backend` will contain the JOSYN-native replacements for all legacy components:

| Future component | Replaces | Notes |
|------------------|----------|-------|
| `JOSYN.Backend.TriggerAgent` | `JobSystem.TriggerAgent` | Polls for scheduled executions |
| `JOSYN.Backend.Scheduling` | `JobSystem.Scheduling` | Time-scheduling logic |
| `JOSYN.Backend.SessionStore` ✅ | `JobSystem.Database` | **Done** — EF Core, SQL Server, `josyn` schema |
| `JOSYN.Backend.GlobalConfig` ✅ | *(new)* | **Done** — `IBootstrapConfig` contract + `FileBootstrapConfig` (file-based INI); supersedes `HardcodedGlobalConfig` |
| `JOSYN.Backend.SessionStarter` ✅ | `JobSystem.SessionStarter` | **Done** — `ISessionStarter`; allocates GUID, persists session, spawns `JAPServer.exe` |
| `JOSYN.Backend.JobRegistry` ✅ | *(replaces implicit company config DB entries)* | **Done** — `IJobRegistry`; `josyn.JobRegistrations` table; per ADR-005 |
| `JOSYN.Backend.ErrorHandler` ✅ | *(new)* | **Done** — `IErrorHandler`, `SqlErrorHandler`, `josyn.ErrorStore`; per ADR-006 |
| `JOSYN.Backend.JobRepository` | `JobSystem.JobRepository` | Resolves job.exe path from job name |
| `JOSYN.Backend.Service` | `JobSystem.Service` | Windows service host |
| `JOSYN.Backend.WorkflowAdapter` | `JobSystem.WorkflowAdapter` | Workflow integration |

---

## Dependencies

```xml
<!-- JOSYN.Backend.JobRegistry: foundation + EF Core (same pattern as SessionStore) -->
<PackageReference Include="JOSYN.Foundation.ResultPattern"          Version="1.0.0-preview01"/>
<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer"  Version="9.0.5"/>

<!-- JOSYN.Backend.SessionStore: foundation + EF Core -->
<PackageReference Include="JOSYN.Foundation.ResultPattern"          Version="1.0.0-preview01"/>
<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer"  Version="9.0.5"/>

<!-- JOSYN.Backend.BootstrapConfig: PropertyBag + ResultPattern -->
<PackageReference Include="JOSYN.Foundation.PropertyBag"            Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Foundation.ResultPattern"          Version="1.0.0-preview01"/>

<!-- JOSYN.Backend.SessionStarter: depends on SessionStore + BootstrapConfig + ResultPattern -->
<PackageReference Include="JOSYN.Backend.BootstrapConfig"  Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Backend.SessionStore"     Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Foundation.ResultPattern" Version="1.0.0-preview01"/>

<!-- JOSYN.Backend.ErrorHandler: foundation + EF Core + Commons.Log -->
<PackageReference Include="JOSYN.Foundation.ResultPattern"          Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Commons.Log"                       Version="1.0.0-preview01"/>
<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer"  Version="9.0.5"/>

<!-- JOSYN.Jap.JAPServer: foundation + josyn-jap shared packages + backend packages -->
<PackageReference Include="JOSYN.Backend.SessionStore"        Version="1.0.0-preview02"/>
<PackageReference Include="JOSYN.Backend.BootstrapConfig"     Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Backend.ErrorHandler"        Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Commons.Log"                 Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Foundation.JIP"              Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Foundation.PropertyBag"      Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Foundation.ResultPattern"    Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Jap.Contract"         Version="1.0.0-preview01"/>
```

Runtime spawn relationships (not NuGet dependencies):
- `josyn-backend` **spawns** `JOSYN.Jap.JAPServer` (built within this repo)
- `JOSYN.Jap.JAPServer` **spawns** `job.exe` (resolved from job repository by JAPServer)

---

## Build & Package

```
.local-build\build.cmd [Release|Debug]               ← builds ALL solutions in the repo
josyn-backend-jap-server\.local-build\build.cmd      ← builds JAPServer only
josyn-backend-bootstrap-config\.local-build\build.cmd ← builds BootstrapConfig only
josyn-backend-session-starter\.local-build\build.cmd ← builds SessionStarter only
josyn-backend-job-registry\.local-build\build.cmd    ← builds JobRegistry only
```

`pack.cmd` at the repo root packs `SessionStore`, `BootstrapConfig`, `SessionStarter`, `JobRegistry`, and `ErrorHandler` to the local feed.
`JAPServer` is not packed.

License: MIT | Company: HAEVG AG | Target: net10.0

---

## Sanity Notes

### Current state — expected

- No test project for `JOSYN.Jap.JAPServer` — known gap, not a violation.
- No test project for `JOSYN.Backend.SessionStore` — known gap, not a violation.
- No test project for `JOSYN.Backend.GlobalConfig` (`BootstrapConfig`) — known gap, not a violation.
- No test project for `JOSYN.Backend.SessionStarter` — known gap, not a violation.
- `HardcodedGlobalConfig` uses compile-time developer-machine constants — superseded by `FileBootstrapConfig`; no longer present.
- `SessionStarter`: on `Process.Start` failure after session is saved, an orphaned row with empty `Result` remains in the store — known limitation; requires a session status column to fix.
- The planned components (`TriggerAgent`, `Scheduling`, etc.) do not exist yet — expected.

### Structural specifics

- **Pattern B repo**: each solution lives in its own kebab sub-folder (`josyn-backend-jap-server/`). See `architecture/repo-structure-conventions.md`.
- **Solution boundary rule**: a project in one solution must not take a project reference to a project in a different solution within this repo. Cross-solution dependencies go via NuGet. Any cross-solution project reference is a violation.
- `JOSYN.Jap.JAPServer` is a Console EXE — must **not** have `GenerateDocumentationFile` or NuGet packaging metadata.

### Dependency constraints

- `JOSYN.Jap.JAPServer`: permitted references are `JOSYN.Foundation.JIP`, `JOSYN.Foundation.PropertyBag`, `JOSYN.Foundation.ResultPattern`, `JOSYN.Jap.Contract`, `JOSYN.Commons.Log`, `JOSYN.Backend.SessionStore`, `JOSYN.Backend.BootstrapConfig`, `JOSYN.Backend.ErrorHandler`. Any other cross-repo reference is a violation.
- Neither current nor future projects in this repo may reference `josyn-job-host` packages.
- Runtime spawning of `JAPServer.exe` (built within this repo) is correct — this is **not** a NuGet dependency.

### Known exceptions (not violations)

- `JAPServer.cs` uses `sealed class` (not `static`) — correct; it has instance state implementing `IJosynApplicationProtocol`.
- `JOSYN.Jap.JAPServer` namespace says "Jap" but the assembly lives in `josyn-backend` — intentional per ADR-004 Challenge 5 rebuttal.
