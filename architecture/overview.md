# Architecture Overview

## Runtime Flow

A job execution follows this sequence:

```
josyn-backend (SessionStarter)
    │
    │  1. Detect scheduled execution (TriggerAgent / Service / WorkflowAdapter)
    │  2. Assign fresh session GUID; write session record to session store
    │  3. Spawn: JAPServer.exe JOSYN-IPC <sessionGUID>
    │  4. Spawn: job.exe       JOSYN-IPC <sessionGUID>
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
josyn-backend (stub)
└── JOSYN.Backend.SessionStarter      ← session lifecycle: GUID, session store, process spawning

josyn-foundation (NuGet packages)
├── JOSYN.Foundation.ResultPattern        ← Result<T>, Error, CallerInfo
├── JOSYN.Foundation.PropertyBag          ← record serialization (INI / JSON)
└── JOSYN.Foundation.JIP                  ← Named Pipe transport + JIP convention layer
        ├── Transport:  PipesClient, PipesServer, PipesProtocol
        └── Convention: JipClient, JipServer, JipDispatcher, Request, Response

josyn-system (NuGet packages + EXE)
├── JOSYN.System.Shared.Contract          ← IJosynApplicationProtocol, ErrorReport
├── JOSYN.System.Shared.Log               ← LocalLog (static, file-based, caller-aware)
└── JOSYN.System.JAPServer (EXE)
        ├── Host.cs                       ← lifecycle, JipDispatcher wiring
        └── JAPServer.cs                  ← IJosynApplicationProtocol implementation

josyn-job-host (NuGet library)
└── JOSYN.System.JobHost
        ├── Core.cs                       ← ICore, entry point, error routing
        ├── JobInvoker.cs                 ← reflection dispatch
        ├── JAPClient.cs                  ← IJosynApplicationProtocol over Named Pipes
        └── Attributes/                   ← [JobEntryPoint], [BeforeJobEntryPoint], etc.

josyn-platform (this repo)
└── architecture, decisions, documentation
```

---

## Dependency Chain

### Code (NuGet) dependencies

```
JOSYN.Foundation.ResultPattern          (no dependencies)
        ▲               ▲               ▲
        │               │               │
JOSYN.Foundation.     JOSYN.Foundation. JOSYN.Backend.
PropertyBag           JIP               SessionStarter  (stub — ResultPattern only)
        ▲               ▲
        └───────┬────────┘
                │
        JOSYN.System.Shared.Contract    (+ ResultPattern)
        JOSYN.System.Shared.Log         (+ ResultPattern)
                ▲
                │
        JOSYN.System.JAPServer          (+ JIP + PropertyBag + Contract + Log)
                │
                │  (protocol consumer — not a code dependency)
                ▼
        JOSYN.System.JobHost            (+ JIP + PropertyBag + Contract + Log)
```

### Runtime (spawn) relationships — not code imports

```
josyn-backend ──spawns──► JOSYN.System.JAPServer.exe  (JOSYN-IPC <guid>)
josyn-backend ──spawns──► job.exe                      (JOSYN-IPC <guid>)
```

`josyn-system` and `josyn-job-host` are **architecturally symmetric**: both consume the same
foundation packages and speak the same protocol. They never reference each other.
`josyn-backend` never takes a NuGet dependency on `josyn-system`; the coupling is the GUID.

---

## IPC Protocol (JIP — JOSYN Inter-Process)

Communication uses **two unidirectional named pipes** per session:
- `JOSYN-REQ-<sessionGUID>` — client writes, server reads
- `JOSYN-RSP-<sessionGUID>` — server writes, client reads

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
job.exe                          JAPServer
  │                                  │
  ├─ IPC fails                        │
  │   └─ LocalLog.Error (local)       │
  │                                   │
  ├─ Job fails (IPC OK)               │
  │   ├─ LocalLog.Error (local)       │
  │   └─ PutError(ErrorReport) ──────►│
  │                                   ├─ PropertyBag.Deserialize<ErrorReport>
  │                                   └─ LocalLog.Error(report.Causer, ...)
  │
  └─ PutError itself fails
      └─ LocalLog.Error fallback (local only)
```

`ErrorReport` fields: `Causer` (process name), `Message`, `CallStack`, `ExceptionDetails`, `OccurredAt`.

---

## Logging

`LocalLog` (from `JOSYN.System.Shared.Log`) is a static, process-local file logger:

```csharp
LocalLog.LogDirectory = "...";         // set at startup
LocalLog.EnableConsoleOutput = true;   // DEBUG builds only

LocalLog.Info("message");
LocalLog.Error("message");
LocalLog.Error(result);                // overload for Result
LocalLog.Error(causer, "message");     // writes to <LogDir>/<causer>/<date>.log
```

I/O errors in the logger are silently swallowed — it never crashes the host.
