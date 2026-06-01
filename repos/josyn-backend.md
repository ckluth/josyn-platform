# josyn-backend

**Role:** Scheduler and session-orchestration layer — the integration bridge between the
legacy job backend and the JOSYN platform. Also the home of `JOSYN.Jap.JAPServer`
(relocated from `josyn-jap` per ADR-004).

**Location:** `C:\Users\chris\OneDrive\DevGit\josyn-backend`
**Version:** `1.0.0-preview01` (stub)

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

## Current state — stub + JAPServer (relocated from josyn-jap per ADR-004)

`JOSYN.Backend.SessionStarter` is a compilable placeholder. `JOSYN.Jap.JAPServer` was
relocated here from `josyn-jap` and has its own solution file.

```
josyn-backend/
├── JOSYN.Backend.SessionStarter.slnx       ← library solution
├── JOSYN.Backend.SessionStarter/
│
├── JOSYN.Backend.SessionStarter.Mock.slnx  ← mock EXE solution (local testing)
├── JOSYN.Backend.SessionStarter.Mock/
│
├── JOSYN.Jap.JAPServer.slnx                ← relocated EXE solution (from josyn-jap)
├── JOSYN.Jap.JAPServer/
│
└── .local-build/
    └── build.cmd                           ← builds ALL solutions
```

### JOSYN.Backend.SessionStarter

**Purpose:** The migration rendezvous. In the new system, this component:

1. Receives a job-start request (from a service, trigger, or workflow adapter)
2. Assigns a fresh `Guid` as the session key
3. Writes a new session record into the session store
4. Spawns `JAPServer.exe JOSYN-IPC <sessionGUID>` — starts the per-session JAP server (JAPServer then spawns `job.exe`)
5. Returns the session GUID to the caller

**Dependencies:** `JOSYN.Foundation.ResultPattern` only

```csharp
public interface ISessionStarter
{
    static abstract Result<Guid> StartSession(StartSessionRequest request);
}

public sealed record StartSessionRequest(string JobName);

public static class SessionStarter : ISessionStarter
{
    public static Result<Guid> StartSession(StartSessionRequest request) =>
        new Error("Noch nicht implementiert.");
}
```

---

## JOSYN.Jap.JAPServer

**Purpose:** Per-session JAP protocol server executable. Listens on named pipes (JIP), receives
JAP requests from job executables, dispatches to the `IJosynApplicationProtocol` implementation.
Relocated from `josyn-jap` per ADR-004 — it needs backend resources (`SessionStore`,
`CompanyConfig`) that are owned by this repo.

**Dependencies:** `JOSYN.Foundation.JIP`, `JOSYN.Foundation.PropertyBag`,
`JOSYN.Foundation.ResultPattern`, `JOSYN.Jap.Shared.Contract`, `JOSYN.Jap.Shared.Log`

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
        // 2. Create JAPServer instance
        // 3. Register all IJosynApplicationProtocol methods with JipDispatcher
        // 4. Start PipesServer — blocking until ESC or fatal error
    }
}

// JAPServer.cs — IJosynApplicationProtocol implementation
sealed class JAPServer : IJosynApplicationProtocol
{
    Task<Result<string>> GetRawArguments();        // PoC: returns fake INI data
    Task<Result> PutRawResult(string result);      // PoC: prints to console
    Task<Result> PutError(string serializedError); // deserializes ErrorReport → LocalLog
}
```

### JIP dispatcher wiring

```csharp
_jipDispatcher.RegisterAll<IJosynApplicationProtocol>(jAPServer);
```

`RegisterAll` auto-discovers all methods on `IJosynApplicationProtocol` via reflection and maps them to `what` strings matching method names.

### PoC limitations

- `GetRawArguments` returns hardcoded fake INI data (`FakeReadArgumentsFromFile`)
- Demo session key: `dea5611d-d740-437f-ad93-7a5dc5ae4299` in `launchSettings.json`
- Machine-specific build output path in `Directory.Build.props`

---

## Planned components

When migrated, `josyn-backend` will contain the JOSYN-native replacements for all legacy components:

| Future component | Replaces | Notes |
|------------------|----------|-------|
| `JOSYN.Backend.TriggerAgent` | `JobSystem.TriggerAgent` | Polls for scheduled executions |
| `JOSYN.Backend.Scheduling` | `JobSystem.Scheduling` | Time-scheduling logic |
| `JOSYN.Backend.SessionStore` | `JobSystem.Database` | Session persistence (EF Core; own SQL context) |
| `JOSYN.Backend.JobRepository` | `JobSystem.JobRepository` | Resolves job.exe path from job name |
| `JOSYN.Backend.SessionStarter` | `JobSystem.SessionStarter` | ✅ Stub exists |
| `JOSYN.Backend.Service` | `JobSystem.Service` | Windows service host |
| `JOSYN.Backend.WorkflowAdapter` | `JobSystem.WorkflowAdapter` | Workflow integration |

---

## Dependencies

```xml
<!-- JOSYN.Backend.SessionStarter: ResultPattern only -->
<PackageReference Include="JOSYN.Foundation.ResultPattern" Version="1.0.0-preview01"/>

<!-- JOSYN.Jap.JAPServer: foundation + josyn-jap shared packages -->
<PackageReference Include="JOSYN.Foundation.JIP"              Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Foundation.PropertyBag"      Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Foundation.ResultPattern"    Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Jap.Shared.Contract"         Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Jap.Shared.Log"              Version="1.0.0-preview01"/>
```

Runtime spawn relationships (not NuGet dependencies):
- `josyn-backend` **spawns** `JOSYN.Jap.JAPServer` (built within this repo)
- `JOSYN.Jap.JAPServer` **spawns** `job.exe` (resolved from job repository by JAPServer)

---

## Build & Package

```
.local-build\build.cmd [Release|Debug]   ← builds ALL solutions in the repo
.local-build\pack.cmd                    ← outputs SessionStarter to ..\..\local-packages\
```

Each solution can also be built independently via its own `dotnet build`.

License: MIT | Company: HAEVG AG | Target: net10.0

---

## Sanity Notes

### Current state — expected

- All methods in `SessionStarter` return `new Error("Noch nicht implementiert.")` — correct PoC stub state, not a violation.
- No test project for `SessionStarter` yet — known gap in the current PoC phase, not a violation.
- `SessionStarter.Mock` solution does not yet exist — expected; it is the planned next step.
- The planned components (`TriggerAgent`, `Scheduling`, `SessionStore`, etc.) do not exist yet — expected.

### Structural specifics

- **Multi-solution repo**: three solutions (`SessionStarter`, `SessionStarter.Mock`, `JAPServer`), each in its own `.slnx` file. `.local-build/build.cmd` builds all.
- **Solution boundary rule**: a project in one solution must not take a project reference to a project in a different solution within this repo. Cross-solution dependencies go via NuGet. Any cross-solution project reference is a violation.
- `JOSYN.Backend.SessionStarter` is a NuGet library — must have `GenerateDocumentationFile`, NuGet metadata, `icon.png`.
- `JOSYN.Jap.JAPServer` is a Console EXE — must **not** have `GenerateDocumentationFile` or NuGet packaging metadata.

### Dependency constraints

- `JOSYN.Backend.SessionStarter`: only `JOSYN.Foundation.ResultPattern` is permitted.
- `JOSYN.Jap.JAPServer`: permitted references are `JOSYN.Foundation.JIP`, `JOSYN.Foundation.PropertyBag`, `JOSYN.Foundation.ResultPattern`, `JOSYN.Jap.Shared.Contract`, `JOSYN.Jap.Shared.Log`. Any other cross-repo reference is a violation.
- Neither project may reference `josyn-job-host` packages.
- Runtime spawning of `JAPServer.exe` (within this repo) is correct — this is **not** a NuGet dependency of `SessionStarter`. `JAPServer` in turn spawns `job.exe` (from job repository).

### Known exceptions (not violations)

- `JAPServer.cs` uses `sealed class` (not `static`) — correct; it has instance state implementing `IJosynApplicationProtocol`.
- `GetRawArguments()` returns hardcoded fake INI data — known PoC limitation, not a violation in the current phase.
- Demo session key in `launchSettings.json` — PoC convenience, not a violation.
- `JOSYN.Jap.JAPServer` namespace says "Jap" but the assembly lives in `josyn-backend` — intentional per ADR-004 Challenge 5 rebuttal; the namespace reflects what the component *is*, not where it lives.

