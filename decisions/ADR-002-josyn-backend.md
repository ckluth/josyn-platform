# ADR-002 — Introducing `josyn-backend` as a Separate Layer

**Date:** 2026-05-29
**Status:** Accepted

---

## Context

The current JOSYN platform PoC has four repos:

| Repo | Role |
|------|------|
| `josyn-foundation` | Infrastructure primitives |
| `josyn-jap` | Per-session JAP protocol server + shared packages |
| `josyn-job-host` | Job executable runtime library |
| `josyn-platform` | Architecture, decisions, cross-cutting docs |

**What is missing:** the layer that *triggers* job sessions. Currently the PoC documents a
"Job Scheduler" as the actor that spawns processes, but no JOSYN repo embodies it.
In the legacy system this role is fulfilled by `JobSystem.Service`,
`JobSystem.TriggerAgent`, `JobSystem.SessionStarter`, and several companion components.

The legacy `SessionStarter` is the key migration seam. It:
1. Creates a session record in the `JobSessions` SQL table
2. Calls `Process.Start(JobHost.exe, sessionUID)`

`JobHost.exe` is the architectural bottleneck of the legacy system: it loads job DLLs via
heavy reflection into its own process space, cannot tolerate conflicting assembly versions,
and must be rebuilt for every .NET target. This wall prevents evolution.

---

## Decision

Introduce **`josyn-backend`** as the fifth repo, representing the scheduling and
session-orchestration layer.

### Responsibility boundaries

| Layer | Owns |
|-------|------|
| `josyn-backend` | Scheduling, trigger evaluation, session-store persistence, session GUID assignment, spawning JAPServer and job.exe |
| `josyn-jap` | The per-session JAP protocol server; receiving and fulfilling `GetRawArguments`, `PutRawResult`, `PutError` |
| `josyn-job-host` | The runtime embedded in job executables; protocol client, argument deserialization, reflection dispatch |
| `josyn-foundation` | Infrastructure primitives shared by all layers |

`josyn-jap` and `josyn-job-host` are **unaware of `josyn-backend`**.
The session GUID is the only coupling — passed as a CLI argument at spawn time.

### The migration seam

`SessionStarter` (in `josyn-backend`) replaces the `JobSystem.SessionStarter` call site:

```
Old:  SessionStarter → Process.Start(JobHost.exe, <sessionUID>)

New:  SessionStarter → Process.Start(JAPServer.exe, JOSYN-IPC <guid>)
      SessionStarter → Process.Start(job.exe,       JOSYN-IPC <guid>)
```

This swap is the minimum viable change to move a job from the legacy model to the JOSYN model.
Old and new can run in parallel: legacy jobs continue via the old `SessionStarter`;
migrated jobs use the new `SessionStarter`.

### Stub-first approach

`josyn-backend` is introduced as a compilable stub today:
- `JOSYN.Backend.SessionStarter` defines the contract (`ISessionStarter`) and a
  placeholder implementation that returns `Result.Error("Noch nicht implementiert.")`
- All other planned components (`TriggerAgent`, `Scheduling`, `SessionStore`, `JobRepository`,
  `Service`, `WorkflowAdapter`) are deferred until migration begins

---

## Consequences

- `josyn-jap` is **no longer called "the backend"** in platform documentation.
  The correct term for `josyn-jap` is "the JAP session server" or "the per-session server".
  "Backend" now refers exclusively to `josyn-backend`.
- The platform vocabulary gains a new entry: **Backend** → `josyn-backend`
- The platform now has five repos; `josyn-platform/README.md` reflects this
- The runtime flow in `architecture/overview.md` is updated to show `josyn-backend`
  as the spawner of both `JAPServer.exe` and `job.exe`
- `josyn-backend` depends only on `JOSYN.Foundation.ResultPattern` in the stub phase.
  It will **never** take a NuGet dependency on `josyn-jap`; the relationship is
  purely runtime (spawn) — not a code import
