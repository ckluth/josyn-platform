# josyn-backend

**Role:** Scheduler and session-orchestration layer — the integration bridge between the
legacy job backend and the JOSYN platform. Also the home of `JOSYN.Jap.JAPServer`
(relocated from `josyn-jap` per ADR-004).

**Location:** `C:\DevGit\josyn-backend`
**Version:** `1.0.0-preview01` (stub)

---

## Decisions

Backend-specific architectural decisions are recorded in the consolidated
[`decisions/`](../../decisions/) folder in `josyn-platform`.

| ADR | Title |
|-----|-------|
| [ADR-005B-01](../../decisions/ADR-005B-01-backend-building-block-model.md) | Backend Building Block Model |
| [ADR-006B-01](../../decisions/ADR-006B-01-session-store.md) | SessionStore |
| [ADR-006B-02](../../decisions/ADR-006B-02-global-config.md) | GlobalConfig |
| [ADR-007B-02](../../decisions/ADR-007B-02-job-registry.md) | JobRegistry |
| [ADR-011B-01](../../decisions/ADR-011B-01-error-handler.md) | ErrorHandler |
| [ADR-017B-01](../../decisions/ADR-017B-01-session-starter-relocation.md) | Session-Starter Relocation into JAPServer |
| [ADR-018B-01](../../decisions/ADR-018B-01-job-start-negotiation.md) | Job Start Negotiation (Accept / Reject) |
| [ADR-017B-02](../../decisions/ADR-017B-02-orphaned-sessions.md) | Resolving Orphaned Sessions |
| [ADR-017B-03](../../decisions/ADR-017B-03-credential-provider.md) | ICredentialProvider (Impersonation Extension Point) |

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
| What backend spawns | `JobHost.exe <sessionUID>` | `JAPServer.exe JOSYN-START @<path>` |
| What JAPServer spawns | *(n/a — JobHost loaded job via reflection)* | `job.exe JOSYN-IPC <guid>` |
| Job loading | Reflection into JobHost process space | job.exe is an independent process |
| .NET versioning | One JobHost per .NET version | Each job.exe targets its own .NET version freely |
| Backend coupling | JobHost knew the session-store schema | JAPServer owns the protocol; job.exe knows nothing of the backend |

Old and new can coexist: jobs migrated to the JOSYN model are started via the new
`SessionLauncher`; legacy jobs continue to be started via the old `SessionStarter` in parallel.

---

## Current state

`JOSYN.Jap.JAPServer` is a working EXE, relocated from `josyn-jap` per ADR-004.
`JOSYN.Backend.SessionStore` is the first real Category A NuGet package, ported from the
sandbox prototype per ADR-006B-01. (The sandbox is now called `josyn-playground`.)
`JOSYN.Backend.BootstrapConfig` is the second Category A NuGet package; provides `IBootstrapConfig`
and `FileBootstrapConfig` (reads `josyn.bootstrap.ini`), per ADR-006B-02.
`JOSYN.Backend.SessionLauncherContract` is a Category A NuGet package; provides `SessionStartRequest` — the shared contract for the `JOSYN-START` session-start protocol (ADR-017B-01).
`JOSYN.Backend.SessionLauncher` is a Category A NuGet package; provides `ISessionLauncher` / `SessionLauncher` — the orchestrator-side thin launcher (ADR-017B-01).
`JOSYN.Jap.JAPServer` supports a single invocation mode: `JOSYN-START @<path>` — reads the `SessionStartRequest` temp file, runs `Turnstile`, persists the session, and proceeds to the JAP serve loop.

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
├── josyn-backend-session-starter/          ← REMOVED (superseded by SessionLauncher per ADR-017B-01)
│   ├── nuget.config
│   ├── JOSYN.Backend.SessionStarter.slnx
│   ├── .local-build/                       ← solution-local build + pack scripts
│   └── JOSYN.Backend.SessionStarter/
├── josyn-backend-session-launcher-contract/ ← Pattern B sub-folder
│   ├── nuget.config
│   ├── JOSYN.Backend.SessionLauncherContract.slnx
│   ├── .local-build/                       ← solution-local build + pack scripts
│   └── JOSYN.Backend.SessionLauncherContract/
├── josyn-backend-session-launcher/         ← Pattern B sub-folder
│   ├── nuget.config
│   ├── JOSYN.Backend.SessionLauncher.slnx
│   ├── .local-build/                       ← solution-local build + pack scripts
│   └── JOSYN.Backend.SessionLauncher/
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
`JOSYN.Commons.Helpers`, `JOSYN.Backend.SessionStore`, `JOSYN.Backend.ErrorHandler`,
`JOSYN.Backend.SessionLauncherContract`

**Type:** `net10.0` Console EXE

### CLI contract

```
JOSYN.Jap.JAPServer.exe JOSYN-START @<path>
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
| `JOSYN.Backend.SessionStarter` ❌ | `JobSystem.SessionStarter` | **Removed** — replaced by `SessionLauncherContract` + `SessionLauncher` (ADR-017B-01) |
| `JOSYN.Backend.SessionLauncherContract` ✅ | *(new)* | **Done** — `SessionStartRequest` shared contract; consumed by both `SessionLauncher` and `JAPServer` (ADR-017B-01) |
| `JOSYN.Backend.SessionLauncher` ✅ | `JobSystem.SessionStarter` | **Done** — `ISessionLauncher`; validates job, resolves `TechnicalUserName`, serializes to temp file, spawns `JAPServer JOSYN-START @<path>` (ADR-017B-01) |
| `JOSYN.Backend.JobRegistry` ✅ | *(replaces implicit company config DB entries)* | **Done** — `IJobRegistry`; `josyn.JobRegistrations` table; per ADR-007B-02 |
| `JOSYN.Backend.ErrorHandler` ✅ | *(new)* | **Done** — `IErrorHandler`, `SqlErrorHandler`, `josyn.ErrorStore`; per ADR-011B-01 |
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

<!-- JOSYN.Backend.SessionStarter removed (ADR-017B-01) -->

<!-- JOSYN.Backend.ErrorHandler: foundation + EF Core + Commons.Log -->
<PackageReference Include="JOSYN.Foundation.ResultPattern"          Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Commons.Log"                       Version="1.0.0-preview01"/>
<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer"  Version="9.0.5"/>

<!-- JOSYN.Backend.SessionLauncherContract: ResultPattern only -->
<PackageReference Include="JOSYN.Foundation.ResultPattern"          Version="1.0.0-preview01"/>

<!-- JOSYN.Backend.SessionLauncher: contract + PropertyBag + BootstrapConfig + JobRegistry -->
<PackageReference Include="JOSYN.Backend.SessionLauncherContract"  Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Backend.BootstrapConfig"          Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Backend.JobRegistry"              Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Foundation.PropertyBag"           Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Foundation.ResultPattern"         Version="1.0.0-preview01"/>

<!-- JOSYN.Jap.JAPServer: foundation + josyn-jap shared packages + backend packages -->
<PackageReference Include="JOSYN.Backend.SessionStore"        Version="1.0.0-preview02"/>
<PackageReference Include="JOSYN.Backend.BootstrapConfig"     Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Backend.ErrorHandler"        Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Backend.SessionLauncherContract" Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Commons.Log"                 Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Commons.Helpers"             Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Foundation.JIP"              Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Foundation.PropertyBag"      Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Foundation.ResultPattern"    Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Jap.Contract"         Version="1.0.0-preview01"/>
```

Runtime spawn relationships (not NuGet dependencies):
- `josyn-backend` **spawns** `JOSYN.Jap.JAPServer` (built within this repo)
- `JOSYN.Jap.JAPServer` **spawns** `job.exe` (resolved from job repository by JAPServer)

---

### Build & Package

```
.local-build\build.cmd [Release|Debug]               ← builds ALL solutions in the repo
josyn-backend-jap-server\.local-build\build.cmd      ← builds JAPServer only
josyn-backend-bootstrap-config\.local-build\build.cmd ← builds BootstrapConfig only
josyn-backend-job-registry\.local-build\build.cmd    ← builds JobRegistry only
```

`pack.cmd` at the repo root packs `SessionStore`, `BootstrapConfig`, `SessionLauncher`, `JobRegistry`, and `ErrorHandler` to the local feed.
`JAPServer` is not packed.

License: MIT | Company: HAEVG AG | Target: net10.0

---

## Sanity Notes

### Current state — expected

- No test project for `JOSYN.Jap.JAPServer` — known gap, not a violation.
- No test project for `JOSYN.Backend.SessionStore` — known gap, not a violation.
- No test project for `JOSYN.Backend.GlobalConfig` (`BootstrapConfig`) — known gap, not a violation.
- `HardcodedGlobalConfig` uses compile-time developer-machine constants — superseded by `FileBootstrapConfig`; no longer present.
- `SessionStarter` has been removed — no orphaned-row concern applies (ADR-017B-01).
- The planned components (`TriggerAgent`, `Scheduling`, etc.) do not exist yet — expected.

### Structural specifics

- **Pattern B repo**: each solution lives in its own kebab sub-folder (`josyn-backend-jap-server/`). See `architecture/repo-structure-conventions.md`.
- **Solution boundary rule**: a project in one solution must not take a project reference to a project in a different solution within this repo. Cross-solution dependencies go via NuGet. Any cross-solution project reference is a violation.
- `JOSYN.Jap.JAPServer` is a Console EXE — must **not** have `GenerateDocumentationFile` or NuGet packaging metadata.

### Dependency constraints

- `JOSYN.Jap.JAPServer`: permitted references are `JOSYN.Foundation.JIP`, `JOSYN.Foundation.PropertyBag`, `JOSYN.Foundation.ResultPattern`, `JOSYN.Jap.Contract`, `JOSYN.Commons.Log`, `JOSYN.Commons.Helpers`, `JOSYN.Backend.SessionStore`, `JOSYN.Backend.BootstrapConfig`, `JOSYN.Backend.ErrorHandler`, `JOSYN.Backend.SessionLauncherContract`. Any other cross-repo reference is a violation.
- Neither current nor future projects in this repo may reference `josyn-job-host` packages.
- Runtime spawning of `JAPServer.exe` (built within this repo) is correct — this is **not** a NuGet dependency.

### Known exceptions (not violations)

- `JAPServer.cs` uses `sealed class` (not `static`) — correct; it has instance state implementing `IJosynApplicationProtocol`.
- `JOSYN.Jap.JAPServer` namespace says "Jap" but the assembly lives in `josyn-backend` — intentional per ADR-004 Challenge 5 rebuttal.
