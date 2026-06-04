# JOSYN and its Predecessor

## What is JOSYN?

JOSYN (Job System Next) is a platform for executing scheduled jobs as isolated executable
processes. A central scheduler orchestrates execution: it spawns job processes, passes them
arguments via a named-pipe protocol, and receives results or error reports in return.

JOSYN is a **platform**, not a system in the traditional sense. It provides the infrastructure
and contracts that job developers and operators build upon — it does not prescribe what jobs do.

## The Predecessor: JobSystem

JOSYN replaces the company's existing `JobSystem`. Many of JobSystem's concepts carry forward;
the core execution model is fundamentally changed.

### What JobSystem does

| Component | Responsibility |
|-----------|----------------|
| `JobSystem.Service` | Windows service hosting a WCF server; receives "start job" requests |
| `JobSystem.TriggerAgent` | Windows service; polls once per minute for scheduled executions |
| `JobSystem.Scheduling` | Proprietary time-scheduling mechanism |
| `JobSystem.WorkflowAdapter` | Activates a job as a step within a proprietary workflow |
| `JobSystem.Database` | SQL Server with `JobSessions` table; every session is persisted here |
| `JobSystem.JobRepository` | File-system tree; all job assemblies are deployed here |
| `JobSystem.SessionStarter` | Creates a session row, then spawns `JobHost.exe <sessionUID>` |
| `JobSystem.JobHost` | Loads the job DLL via reflection; one isolated process per session |

### The flaw: shared process space

`JobHost.exe` was the weak point. It loaded job assemblies into its own process via reflection,
required careful assembly-resolve gymnastics, could never use modern EF Core due to version
conflicts, and had to be recompiled for every .NET version. The result: a wall preventing any
real evolution.

### The cure: process-space separation

JOSYN eliminates the shared-process trap. Instead of loading job code into a shared host process,
each job now runs as a fully independent OS process. The scheduler spawns `JAPServer.exe`, which
in turn spawns `job.exe`. The two processes communicate exclusively through a pair of GUID-named
pipes. No shared memory, no shared assembly load context. Each job executable targets any .NET
version it needs freely.

| | Old | New |
|---|-----|-----|
| What backend spawns | `JobHost.exe <sessionUID>` | `JAPServer.exe JOSYN-IPC <guid>` |
| Job loading | Reflection into JobHost process space | `job.exe` is an independent process |
| .NET versioning | One JobHost per .NET version | Each `job.exe` targets its own version freely |
| Backend coupling | JobHost knew the session-store schema | `JAPServer` owns the protocol; `job.exe` knows nothing of the backend |

## Coexistence During Migration

Old and new can run in parallel. Jobs migrated to the JOSYN model are started via the new
`SessionStarter`; legacy jobs continue via the old `JobSystem.SessionStarter`. The clean
seam is `SessionStarter` — changing what it spawns is all that is needed for a phased
migration, one job at a time.

## Stakeholders

| Role | Description |
|------|-------------|
| **Job developers** | Authors of job executables; they link `JOSYN.JobHost` and implement `[JobEntryPoint]` |
| **Operators** | Install, deploy, and maintain the backend on server machines |
| **Schedulers / workflow owners** | Configure when jobs run via trigger agent or workflow adapter |
| **Auditors** | Verify job execution history via the session store |

## External Connections

| System | Role |
|--------|------|
| **SQL Server** | Session store database; records all job sessions |
| **Active Directory** | Domain user identities used for job impersonation |
| **Company workflow system** | Triggers job execution as workflow steps via `WorkflowAdapter` |
| **Company config-manager** | Currently holds the job registry (job names, schedules, impersonation users) |
