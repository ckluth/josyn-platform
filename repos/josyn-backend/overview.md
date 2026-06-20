# josyn-backend

**Role:** Scheduler and session-orchestration layer — the integration bridge between the
legacy job backend and the JOSYN platform. Provides NuGet library packages consumed by
`josyn-session-broker` and future orchestration components.

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
| [ADR-017B-03](../../decisions/ADR-017B-03-credential-provider.md) | IdentityAdapter (Impersonation Extension Point) |
| [ADR-020](../../decisions/ADR-020-company-adapter-model.md) | Company Adapter Model (Out-of-Process) |
| [ADR-021](../../decisions/ADR-021-impersonated-process-launch.md) | Impersonated Process Launch for job.exe |
| [ADR-022](../../decisions/ADR-022-interactive-job-launch.md) | Headless / Interactive Distinction for job.exe Launch |
| [ADR-024](../../decisions/ADR-024-ticker.md) | The Ticker: Periodic Process Launcher |
| [ADR-026](../../decisions/ADR-026-schedule-definition-language.md) | Schedule Definition Language |
| [ADR-027](../../decisions/ADR-027-job-schedule-store.md) | JobSchedule Store |
| [ADR-028](../../decisions/ADR-028-argument-records.md) | ArgumentRecord: Named Argument Payloads in the Job Registry |
| [ADR-029](../../decisions/ADR-029-timescheduler-evaluation-strategy.md) | TimeScheduler Evaluation Strategy: Tolerance Window and Fired-Slot Log |

---

## Architectural position

`josyn-backend` is the **outermost layer** of the JOSYN platform. It owns:

- **Scheduling** — detecting when a job should run (polling, triggers, workflow activation)
- **Session lifecycle** — creating session entries in the session store, assigning GUIDs
- **Process spawning** — launching `SessionBroker.exe` with the session GUID; SessionBroker in turn spawns `job.exe`
- **Result collection** — reading session completion status from the session store

`josyn-jap` (shared contracts) and `josyn-job-host` (job executable runtime) are unaware of
`josyn-backend`. `josyn-session-broker` consumes backend NuGet packages.
The session GUID is the only runtime coupling: `josyn-backend` spawns `SessionBroker.exe`;
`SessionBroker` forwards it to `job.exe` when it spawns it.

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

### The cure — "SessionLauncher spawns SessionBroker" eliminates the shared-process trap

`SessionStarter` is already the clean seam between the backend and the execution engine.
Changing what it spawns is all that is needed for a phased migration.

The real architectural cure is **process-space separation via the JIP trick**: instead of loading
job code into a shared host process (the old `JobHost`), each job now runs as a fully independent
OS process. The two processes — `SessionBroker.exe` and `job.exe` — communicate exclusively through
a pair of GUID-named pipes (`JOSYN.Foundation.JIP`). No shared memory, no shared assembly load
context. Each process targets any .NET version it needs. Session isolation is structural, not
enforced by code conventions.

| | Old | New |
|---|-----|-----|
| What backend spawns | `JobHost.exe <sessionUID>` | `SessionBroker.exe JOSYN-START @<path>` |
| What SessionBroker spawns | *(n/a — JobHost loaded job via reflection)* | `job.exe JOSYN-IPC <guid>` |
| Job loading | Reflection into JobHost process space | job.exe is an independent process |
| .NET versioning | One JobHost per .NET version | Each job.exe targets its own .NET version freely |
| Backend coupling | JobHost knew the session-store schema | SessionBroker owns the protocol; job.exe knows nothing of the backend |

Old and new can coexist: jobs migrated to the JOSYN model are started via the new
`SessionLauncher`; legacy jobs continue to be started via the old `SessionStarter` in parallel.

---

## Current state

`JOSYN.SessionBroker` has been renamed from `JOSYN.Jap.JAPServer` and extracted to `josyn-session-broker` per ADR-025.
`JOSYN.Backend.SessionStore` is the first real Category A NuGet package, ported from the
sandbox prototype per ADR-006B-01. (The sandbox is now called `josyn-playground`.)
`JOSYN.Backend.BootstrapConfig` is the second Category A NuGet package; provides `IBootstrapConfig`
and `FileBootstrapConfig` (reads `josyn.bootstrap.ini`), per ADR-006B-02.
`JOSYN.Backend.Contracts` is a Category A NuGet package; provides `SessionBrokerConstants`, `SessionLaunchRequest`, `SessionStartSpec`, `ExecutionStatus`, and `IJobSessionRecord` — the shared contracts consumed by `SessionLauncher` and `SessionBroker` (consolidates what was previously `SessionLauncherContract`, ADR-017B-01).
`JOSYN.Backend.SessionLauncher` is a Category A NuGet package; provides `ISessionLauncher` / `SessionLauncher` — the orchestrator-side thin launcher (ADR-017B-01).
`JOSYN.Backend.JobScheduleStore` is a Category A NuGet package; provides `IJobScheduleStore`, `IJobScheduleEntryRecord`, `IFiredSlotStore`, `SqlJobScheduleStore`, `SqlFiredSlotStore` — scheduling configuration and fired-slot deduplication log (ADR-027, ADR-029).
`JOSYN.Backend.JobRegistry` extended with `ArgumentRecords`: `IArgumentRecord` / `ArgumentRecord`, `IJobRegistry.GetArgument()` — resolves named INI payloads for scheduled launches (ADR-028).
`JOSYN.Backend.TimeScheduler` is a Ticker-target EXE; implements the full tolerance-window + fired-slot-log evaluation algorithm — at-most-once session launch per scheduled slot (ADR-027, ADR-028, ADR-029).

```
josyn-backend/
├── .local-build/
│   ├── build.cmd                               ← builds ALL solutions in the repo
│   └── pack.cmd
├── db/                                         ← JOSYN Storage Realm scripts
│   ├── bootstrap-local-dev.sql                 ← creates DB, schema, dev login (dev only)
│   └── migrations/
│       ├── V001__session_store.sql             ← josyn.SessionStore
│       ├── V002__job_registry.sql              ← josyn.JobRegistry + FK_SessionStore_JobRegistry
│       ├── V003__error_store.sql               ← josyn.ErrorStore
│       ├── V004__config_store.sql              ← josyn.ConfigStore
│       └── V005__session_store_extend.sql      ← SessionStore schema extension
├── josyn-backend-contracts/                    ← Pattern B sub-folder
│   ├── nuget.config
│   ├── JOSYN.Backend.Contracts.slnx
│   ├── .local-build/                           ← solution-local build + pack scripts
│   └── JOSYN.Backend.Contracts/               ← SessionBrokerConstants, SessionLaunchRequest,
│                                               │  SessionStartSpec, ExecutionStatus, IJobSessionRecord
├── josyn-backend-session-store/                ← Pattern B sub-folder
│   ├── nuget.config
│   ├── JOSYN.Backend.SessionStore.slnx
│   ├── .local-build/                           ← solution-local build + pack scripts
│   └── JOSYN.Backend.SessionStore/
├── josyn-backend-config-store/                 ← Pattern B sub-folder
│   ├── nuget.config
│   ├── JOSYN.Backend.ConfigStore.slnx
│   ├── .local-build/                           ← solution-local build + pack scripts
│   └── JOSYN.Backend.ConfigStore/
├── josyn-backend-bootstrap-config/             ← Pattern B sub-folder
│   ├── nuget.config
│   ├── JOSYN.Backend.BootstrapConfig.slnx
│   ├── .local-build/                           ← solution-local build + pack scripts
│   └── JOSYN.Backend.BootstrapConfig/
├── josyn-backend-job-registry/                 ← Pattern B sub-folder
│   ├── nuget.config
│   ├── JOSYN.Backend.JobRegistry.slnx
│   ├── .local-build/                           ← solution-local build + pack scripts
│   └── JOSYN.Backend.JobRegistry/
├── josyn-backend-session-launcher/             ← Pattern B sub-folder
│   ├── nuget.config
│   ├── JOSYN.Backend.SessionLauncher.slnx
│   ├── .local-build/                           ← solution-local build + pack scripts
│   └── JOSYN.Backend.SessionLauncher/
├── josyn-backend-error-handler/                ← Pattern B sub-folder
│   ├── nuget.config
│   ├── JOSYN.Backend.ErrorHandler.slnx
│   ├── .local-build/                           ← solution-local build + pack scripts
│   └── JOSYN.Backend.ErrorHandler/
├── josyn-backend-ticker/                       ← Pattern B sub-folder (EXE)
│   ├── JOSYN.Backend.Ticker.slnx
│   └── .local-build/
├── josyn-backend-listener/                     ← Pattern B sub-folder (EXE)
│   ├── JOSYN.Backend.Listener.slnx
│   └── .local-build/
├── josyn-backend-cli/                          ← Pattern B sub-folder (EXE)
│   ├── JOSYN.Backend.CLI.slnx
│   └── .local-build/
├── josyn-backend-time-scheduler/               ← Pattern B sub-folder (EXE)
│   ├── JOSYN.Backend.TimeScheduler.slnx
│   └── .local-build/
├── josyn-backend-job-schedule-store/           ← Pattern B sub-folder
│   ├── nuget.config
│   ├── JOSYN.Backend.JobScheduleStore.slnx
│   ├── .local-build/                           ← build + pack scripts
│   └── JOSYN.Backend.JobScheduleStore/        ← IJobScheduleStore, IFiredSlotStore,
│                                               │  SqlJobScheduleStore, SqlFiredSlotStore,
│                                               │  Db/ (EF entities + DbContexts)
└── josyn-backend-workflow-runner/              ← Pattern B sub-folder
    ├── WorkflowRunner.slnx
    └── .local-build/
```

---

## Planned components

When migrated, `josyn-backend` will contain the JOSYN-native replacements for all legacy components:

| Future component | Replaces | Notes |
|------------------|----------|-------|
| `JOSYN.Backend.TriggerAgent` | `JobSystem.TriggerAgent` | Polls for scheduled executions — **superseded by `TimeScheduler` + `JobScheduleStore`** |
| `JOSYN.Backend.Scheduling` | `JobSystem.Scheduling` | Time-scheduling logic — **superseded by `JOSYN.Commons.Schedule` (ADR-026) + `TimeScheduler` (ADR-029)** |
| `JOSYN.Backend.SessionStore` ✅ | `JobSystem.Database` | **Done** — EF Core, SQL Server, `josyn` schema |
| `JOSYN.Backend.BootstrapConfig` ✅ | *(new)* | **Done** — `IBootstrapConfig` contract + `FileBootstrapConfig` (file-based INI); provides runtime connection strings and paths (ADR-006B-02) |
| `JOSYN.Backend.SessionStarter` ❌ | `JobSystem.SessionStarter` | **Removed** — replaced by `JOSYN.Backend.Contracts` + `SessionLauncher` (ADR-017B-01) |
| `JOSYN.Backend.Contracts` ✅ | *(new)* | **Done** — `SessionBrokerConstants`, `SessionLaunchRequest`, `SessionStartSpec`, `ExecutionStatus`, `IJobSessionRecord`; shared contracts for `SessionLauncher` and `SessionBroker` (ADR-017B-01) |
| `JOSYN.Backend.SessionLauncher` ✅ | `JobSystem.SessionStarter` | **Done** — `ISessionLauncher`; validates job, resolves `TechnicalUserName`, serializes to temp file, spawns `SessionBroker.exe JOSYN-START @<path>` (ADR-017B-01) |
| `JOSYN.Backend.JobRegistry` ✅ | *(replaces implicit company config DB entries)* | **Done** — `IJobRegistry`; `josyn.JobRegistrations` table; extended with `ArgumentRecords` (ADR-028); per ADR-007B-02 |
| `JOSYN.Backend.ErrorHandler` ✅ | *(new)* | **Done** — `IErrorHandler`, `SqlErrorHandler`, `josyn.ErrorStore`; per ADR-011B-01 |
| `JOSYN.Backend.JobScheduleStore` ✅ | *(new)* | **Done** — `IJobScheduleStore`, `IFiredSlotStore`; `josyn.JobSchedules`, `josyn.JobScheduleEntries`, `josyn.FiredSlots`; per ADR-027, ADR-029 |
| `JOSYN.Backend.TimeScheduler` ✅ | `JobSystem.TriggerAgent` | **Done** — Ticker-target EXE; tolerance-window + fired-slot-log algorithm; at-most-once delivery; per ADR-027, ADR-028, ADR-029 |
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

<!-- JOSYN.Backend.Contracts: ResultPattern only -->
<PackageReference Include="JOSYN.Foundation.ResultPattern"          Version="1.0.0-preview01"/>

<!-- JOSYN.Backend.SessionLauncher: Contracts + PropertyBag + BootstrapConfig + JobRegistry -->
<PackageReference Include="JOSYN.Backend.Contracts"            Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Backend.BootstrapConfig"          Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Backend.JobRegistry"              Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Foundation.PropertyBag"           Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Foundation.ResultPattern"         Version="1.0.0-preview01"/>

```

Runtime spawn relationships (not NuGet dependencies):
- `josyn-backend` (`SessionLauncher`) **spawns** `SessionBroker.exe` (built in `josyn-session-broker`)
- `JOSYN.SessionBroker` **spawns** adapter EXEs (e.g. `Contoso.IdentityAdapter.exe`) with `JOSYN-ADAPTER <guid>`
- `JOSYN.SessionBroker` **spawns** `job.exe` (resolved from job repository by SessionBroker)

---

### Build & Package

```
.local-build\build.cmd [Release|Debug]               ← builds ALL solutions in the repo
josyn-backend-bootstrap-config\.local-build\build.cmd ← builds BootstrapConfig only
josyn-backend-job-registry\.local-build\build.cmd    ← builds JobRegistry only
```

`pack.cmd` at the repo root packs `Contracts`, `SessionStore`, `ConfigStore`, `BootstrapConfig`, `JobRegistry`, `SessionLauncher`, `ErrorHandler`, and `JobScheduleStore` to the shared `local-packages/` feed (sibling directory to this repo).
`Ticker`, `Listener`, `CLI`, `TimeScheduler`, and the other EXE-only subfolders are not packed.

License: MIT | Company: HAEVG AG | Target: net10.0

---

## Sanity Notes

### Current state — expected

- No test project for `JOSYN.Backend.SessionStore` — known gap, not a violation.
- No test project for `JOSYN.Backend.GlobalConfig` (`BootstrapConfig`) — known gap, not a violation.
- No test project for `JOSYN.Backend.JobScheduleStore` — known gap, not a violation.
- No test project for `JOSYN.Backend.TimeScheduler` — known gap, not a violation. The evaluation logic it exercises (`ScheduleEvaluator`) is covered by `JOSYN.Commons.Schedule.Test`.
- `HardcodedGlobalConfig` uses compile-time developer-machine constants — superseded by `FileBootstrapConfig`; no longer present.
- `SessionStarter` has been removed — no orphaned-row concern applies (ADR-017B-01).
- The planned components (`TriggerAgent`, `Scheduling`, etc.) do not exist yet — expected.

### Structural specifics

- **Pattern B repo**: each solution lives in its own kebab sub-folder. See `architecture/repo-structure-conventions.md`.
- **Solution boundary rule**: a project in one solution must not take a project reference to a project in a different solution within this repo. Cross-solution dependencies go via NuGet. Any cross-solution project reference is a violation.

### Dependency constraints

- Neither current nor future projects in this repo may reference `josyn-job-host` packages.
- Runtime spawning of `SessionBroker.exe` (built in `josyn-session-broker`) is correct — this is **not** a NuGet dependency.
