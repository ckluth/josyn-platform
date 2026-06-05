# Architecture Overview

## Runtime Flow

A job execution follows this sequence:

```
josyn-backend (SessionStarter)
    │
    │  1. Detect scheduled execution (TriggerAgent / Service / WorkflowAdapter)
    │  2. Assign fresh session GUID; write session record to session store
    │  3. Spawn: JAPServer.exe JOSYN-IPC <sessionGUID>
    ▼
JOSYN.Jap.JAPServer (in josyn-backend)
    │
    │  4. Spawn: job.exe JOSYN-IPC <sessionGUID>
    ▼
job.exe  ──► Core.Run(args)
    │
    │  5. Parse sessionGUID from args
    │  6. Connect to JAPServer via Named Pipes (session-isolated by GUID)
    ▼
JAPClient  ──► IJosynApplicationProtocol
    │
    │  7. GetRawArguments()  ◄─── JAPServer returns serialized INI/JSON
    ▼
JobInvoker
    │
    │  8. Find [JobEntryPoint] method via reflection
    │  9. PropertyBag.Deserialize(rawArguments, paramType)
    │  10. Invoke method
    ▼
[JobEntryPoint] method  (user-authored business logic)
    │
    │  11. Returns result record (or void)
    ▼
JobInvoker
    │
    │  12. PropertyBag.Serialize(result, IniSerializer)
    │  13. PutRawResult(serialized)  ──► JAPServer stores result
    ▼
job.exe exits with code 0
    │
josyn-backend reads session completion from session store / exit codes
```

### Error paths

```
Any step fails
    │
    ├─ IPC connection failed (step 6)
    │       LocalLog.Error(...)
    │       exit -1
    │
    └─ Job failed (steps 8–13)
            LocalLog.Error(...)
            PutError(ErrorReport)  ──► JAPServer.LocalLog.Error(causer, ...)
            exit -2
```

---

## Component Map

```
josyn-backend (NuGet packages + EXEs)
├── JOSYN.Backend.GlobalConfig         ← runtime config: connection string, exe paths
├── JOSYN.Backend.SessionStore         ← session persistence: EF Core, SQL Server
├── JOSYN.Backend.SessionStarter       ← session lifecycle: GUID, store write, process spawn
├── JOSYN.Backend.JobRegistry          ← job registration: Name, TechnicalUserName; josyn.JobRegistrations table
├── JOSYN.Jap.JAPServer (EXE)          ← relocated from josyn-jap; backed by SessionStore
│       ├── Host.cs                        ← lifecycle, config + store wiring, JipDispatcher
│       └── JAPServer.cs                   ← IJosynApplicationProtocol: reads/writes SessionStore
│
└── ── Future (placeholders) ───────────────────────────────────
    ├── JOSYN.Backend.TriggerAgent     ← replaces JobSystem.TriggerAgent
    ├── JOSYN.Backend.Scheduling       ← replaces JobSystem.Scheduling
    ├── JOSYN.Backend.Service          ← replaces JobSystem.Service (Windows service host)
    ├── JOSYN.Backend.WorkflowAdapter  ← replaces JobSystem.WorkflowAdapter
    ├── listener-service (EXE)         ← future: receives "start job" requests
    ├── ticker-service   (EXE)         ← future: polls for scheduled executions
    └── cli-exe          (EXE)         ← future: operator CLI

josyn-foundation (NuGet packages)
├── JOSYN.Foundation.ResultPattern        ← Result<T>, Error, CallerInfo
├── JOSYN.Foundation.PropertyBag          ← record serialization (INI / JSON)
└── JOSYN.Foundation.JIP                  ← Named Pipe transport + JIP convention layer
        ├── Transport:  PipesClient, PipesServer, PipesProtocol
        └── Convention: JipClient, JipServer, JipDispatcher, Request, Response

josyn-jap (NuGet packages — protocol contracts only)
├── JOSYN.Jap.Shared.Contract          ← IJosynApplicationProtocol, ErrorReport
└── JOSYN.Jap.Shared.Log               ← LocalLog (static, file-based, caller-aware)

josyn-job-host (NuGet library)
└── JOSYN.JobHost
        ├── Core.cs                       ← ICore, entry point, error routing
        ├── JobInvoker.cs                 ← reflection dispatch
        ├── JAPClient.cs                  ← IJosynApplicationProtocol over Named Pipes
        └── Attributes/                   ← [JobEntryPoint], [BeforeJobEntryPoint], etc.

josyn-commons (NuGet packages) — utility satellite; never referenced by josyn-foundation
└── JOSYN.Commons.*                       ← packages added as helpers accumulate (TBD)

josyn-platform (this repo)
└── architecture, decisions, documentation
```

---

## Dependency Chain

### Code (NuGet) dependencies

```
       JOSYN.Foundation.ResultPattern  (no dependencies)
            ▲               ▲
            │               │
       JOSYN.Foundation.   JOSYN.Foundation.
       PropertyBag         JIP
            ▲               ▲
            └───────┬───────┘
                    │
JOSYN.Jap.Shared.Contract (+ ResultPattern)
JOSYN.Jap.Shared.Log      (+ ResultPattern)
        ▲                         ▲
        │                         │
JOSYN.Jap.JAPServer       JOSYN.JobHost
(in josyn-backend          (+ JIP + PropertyBag
 + JIP + PropertyBag        + Contract + Log)
 + SessionStore            [protocol client]
 + GlobalConfig
 + Contract + Log)
[protocol server]
        │                         │
        └── IJosynApplicationProtocol ──┘
             (IPC at runtime — not a code dep.)
```

```
── josyn-backend package chain ────────────────────────────────────────

Backend.GlobalConfig  (no NuGet dependencies)
Backend.SessionStore  (+ ResultPattern)
        ▲                   ▲
        └─────────┬──────────┘
                  │
Backend.SessionStarter  (+ ResultPattern)

Consumers of the backend packages:
  JOSYN.Jap.JAPServer (EXE)                → + SessionStore + GlobalConfig  (see above)

────────────────────────────────────────────────────────────────────────
```

```
── Utility Satellite ──────────────────────────────────────────────────
  JOSYN.Commons.*  (no deps, or ResultPattern only)
          ▲                   ▲                   ▲
          │                   │                   │
  JOSYN.Backend.*     JOSYN.Jap.*         JOSYN.JobHost
  (when needed)       (when needed)       (when needed)

  ✗  JOSYN.Foundation.* never references JOSYN.Commons.*
───────────────────────────────────────────────────────────────────────
```

### Runtime (spawn) relationships — not code imports

```
josyn-backend ──spawns──► JOSYN.Jap.JAPServer.exe  (JOSYN-IPC <guid>)
JOSYN.Jap.JAPServer  ──spawns──► job.exe           (JOSYN-IPC <guid>)
```

`josyn-jap` (shared contracts and logger packages) and `josyn-job-host` speak the same
protocol and consume the same foundation packages — but are not symmetric in the same sense
as before: `josyn-job-host` remains a pure library, while `josyn-jap` is now a contracts-only
repo (the EXE was relocated to `josyn-backend` per ADR-004). They still never reference
each other.

`josyn-backend` takes a NuGet dependency on `JOSYN.Jap.Shared.Contract` and
`JOSYN.Jap.Shared.Log` (via `JOSYN.Jap.JAPServer`). This is intentional and downward:
the contracts repo is a lower-layer package; taking a dependency on it from the backend is
not a layering violation.

---

## IPC Protocol (JIP — JOSYN Inter-Process)

Communication uses **two unidirectional named pipes** per session:
- `req-pipe-<sessionKey>` — client writes, server reads
- `res-pipe-<sessionKey>` — server writes, client reads

Wire format: **length-prefixed binary** (int32 LE + UTF-8 bytes).

At the convention layer (JIP), messages are JSON objects:
```json
// Request
{ "What": "GetRawArguments", "Data": null }

// Response (success)
{ "Succeeded": true, "Data": "<serialized-arguments>" }

// Response (failure)
{ "Succeeded": false, "Error": "Fehlermeldung auf Deutsch" }
```

`JipDispatcher` on the server side auto-discovers handler methods via reflection on `IJosynApplicationProtocol`.
`RegisterAll` supports exactly three method signatures (see ADR-011):

| Signature | Wire encoding |
|-----------|---------------|
| `Task<Result<string>> Method()` | String payload as-is |
| `Task<Result<TEnum>> Method()` where `typeof(TEnum).IsEnum` | `Enum.ToString()` / `Enum.Parse<TEnum>()` |
| `Task<Result> Method(string)` | String payload as-is |

Any other signature causes a hard failure at startup. This rule is closed: `string` and `enum`
are the only supported value types at the JIP convention layer.

Session isolation: each job execution gets a fresh GUID. The scheduler passes this GUID as the first CLI argument (`JOSYN-IPC <guid>`). Both sides use it to name the pipes — zero port conflicts, zero cross-contamination.

---

## Serialization

`PropertyBag` handles record serialization for:
- **Job arguments** (server → job): `GetRawArguments` returns serialized record
- **Job result** (job → server): `PutRawResult` accepts serialized record
- **Error reports** (job → server): `PutError` accepts serialized `ErrorReport`

Supported formats: **INI** (default for arguments/results) and **JSON** (default for `ErrorReport`).

Culture: **`de-DE`** throughout — affects decimal separators, date formats.

Supported property types: `string`, `char`, `bool`, `byte`/`sbyte`/`short`/`ushort`/`int`/`uint`/`long`/`ulong`, `float`/`double`/`decimal`, `DateTime`/`DateTimeOffset`/`DateOnly`/`TimeOnly`/`TimeSpan`, `Guid`, any `enum`. All support `T?` nullable variants.

---

## Error Routing

Three-tier error handling across the platform:

```
job.exe                           JAPServer
  │                                   │
  ├─ IPC fails                        │
  │   └─ LocalLog.Error (local)       │
  │                                   │
  ├─ Job fails (IPC OK)               │
  │   ├─ LocalLog.Error (local)       │
  │   └─ PutError(ErrorReport) ──────►│
  │                                   ├─ PropertyBag.Deserialize<ErrorReport>
  │                                   │─ LocalLog.Error(report.Causer, ...)
  │                                   └─ Propagate error to session store (backend) 
  │
  └─ PutError itself fails
      └─ LocalLog.Error fallback (local only)
```

`ErrorReport` fields: `Causer` (process name), `Message`, `CallStack`, `ExceptionDetails`, `OccurredAt`.

---

## Logging

`LocalLog` (from `JOSYN.Jap.Shared.Log`) is a static, process-local file logger:

```csharp
LocalLog.LogDirectory = "...";         // set at startup
LocalLog.EnableConsoleOutput = true;   // DEBUG builds only

LocalLog.Info("message");
LocalLog.Error("message");
LocalLog.Error(result);                // overload for Result
LocalLog.Error(causer, "message");     // writes to <LogDir>/<causer>/<date>.log
```

I/O errors in the logger are silently swallowed — it never crashes the host.
