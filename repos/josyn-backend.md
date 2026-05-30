# josyn-backend

**Role:** Scheduler and session-orchestration layer — the integration bridge between the
legacy job backend and the JOSYN platform.

**Location:** `C:\Users\chris\OneDrive\DevGit\josyn-backend`
**Version:** `1.0.0-preview01` (stub)

---

## Architectural position

`josyn-backend` is the **outermost layer** of the JOSYN platform. It owns:

- **Scheduling** — detecting when a job should run (polling, triggers, workflow activation)
- **Session lifecycle** — creating session entries in the session store, assigning GUIDs
- **Process spawning** — launching `JAPServer.exe` and `job.exe` with the shared session GUID
- **Result collection** — reading session completion status from the session store

`josyn-jap` (JAPServer) and `josyn-job-host` (job executable runtime) are unaware of
`josyn-backend`. The session GUID is the only coupling — handed by `josyn-backend` to both
processes at spawn time via CLI argument.

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

### The cure — SessionStarter as the migration seam

`SessionStarter` is already the clean seam between the backend and the execution engine.
Changing what it spawns is all that is needed for a phased migration:

| | Old | New |
|---|-----|-----|
| What gets spawned | `JobHost.exe <sessionUID>` | `JAPServer.exe JOSYN-IPC <guid>` + `job.exe JOSYN-IPC <guid>` |
| Job loading | Reflection into JobHost process space | job.exe is an independent process |
| .NET versioning | One JobHost per .NET version | Each job.exe targets its own .NET version freely |
| Backend coupling | JobHost knew the session-store schema | JAPServer knows nothing of josyn-backend |

Old and new can coexist: jobs migrated to the JOSYN model are started via the new
`SessionStarter`; legacy jobs continue to be started via the old `SessionStarter` in parallel.

---

## Current state — stub

Only `JOSYN.Backend.SessionStarter` exists as a compilable placeholder.

```
josyn-backend/
└── JOSYN.Backend.SessionStarter/          ← NuGet library (stub)
    ├── Contracts/
    │   └── ISessionStarter.cs             ← static abstract contract
    ├── SessionStarter.cs                  ← stub; all methods return error
    └── StartSessionRequest.cs             ← session start parameters
```

### JOSYN.Backend.SessionStarter

**Purpose:** The migration rendezvous. In the new system, this component:

1. Receives a job-start request (from a service, trigger, or workflow adapter)
2. Assigns a fresh `Guid` as the session key
3. Writes a new session record into the session store
4. Spawns `JAPServer.exe JOSYN-IPC <sessionGUID>` — starts the per-session JAP server
5. Spawns `job.exe JOSYN-IPC <sessionGUID>` — starts the job process (resolved from the job repository)
6. Returns the session GUID to the caller

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
<!-- Stub: ResultPattern only -->
<PackageReference Include="JOSYN.Foundation.ResultPattern" Version="1.0.0-preview01"/>
```

Runtime spawn relationships (not NuGet dependencies):
- `josyn-backend` **spawns** `JOSYN.Jap.JAPServer` (from `josyn-jap`)
- `josyn-backend` **spawns** `job.exe` (from job repository)

---

## Build & Package

```
.local-build\build.cmd [Release|Debug]
.local-build\pack.cmd    ← outputs to ..\..\local-packages\
```

License: MIT | Company: HAEVG AG | Target: net10.0

---

## Sanity Notes

### Current state — stub (expected)
- All methods in `SessionStarter` return `new Error("Noch nicht implementiert.")` — this is the correct PoC stub state, not a violation.
- No test project exists yet — known gap in the current PoC phase, not a violation.
- The planned components (`TriggerAgent`, `Scheduling`, `SessionStore`, etc.) do not exist yet — expected.

### Dependency constraints
- **Only** `JOSYN.Foundation.ResultPattern` is permitted as a NuGet dependency. Any reference to `josyn-jap`, `josyn-job-host`, or other foundation packages is a violation.
- Runtime spawning of `JAPServer.exe` and `job.exe` is correct — these are **not** NuGet dependencies.

### Structural note
- `josyn-backend` is a single-package repo (stub). Directory structure is simpler than multi-package repos.

