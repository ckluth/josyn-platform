# Services and Executables

## Overview

The JOSYN backend is a set of executables. Some are Windows services (long-running daemons);
some are short-lived per-session processes. Future evolution may include Linux daemons.

## Current State (PoC Phase)

| Component | Type | Purpose |
|-----------|------|---------|
| `JOSYN.Jap.JAPServer.exe` | Console EXE (spawned per session) | Per-session JAP protocol server; bridges backend and job process |
| `JOSYN.Backend.Demo.FakeSessionStarterConsumer.exe` | Console EXE (PoC demo only) | Proves end-to-end session round-trip; not a production component |

## Planned Components (Replacing JobSystem)

| Future Component | Replaces | Notes |
|------------------|----------|-------|
| `JOSYN.Backend.Service` | `JobSystem.Service` (WCF server) | Windows service; receives "start job" requests |
| `JOSYN.Backend.TriggerAgent` | `JobSystem.TriggerAgent` | Windows service; polls once per minute for scheduled executions |
| `JOSYN.Backend.Scheduling` | `JobSystem.Scheduling` | Time-scheduling logic |
| `JOSYN.Backend.WorkflowAdapter` | `JobSystem.WorkflowAdapter` | Activates a job as a step in a proprietary workflow |
| listener-service EXE | — | Future: receives "start job" requests |
| ticker-service EXE | — | Future: polls for scheduled executions |
| operator CLI EXE | — | Future: command-line tool for operators |

## Per-Session: JAPServer

Each job execution spawns a fresh `JAPServer.exe` instance with a unique session GUID:

```
josyn-backend (SessionStarter)
    └──spawns──► JAPServer.exe JOSYN-IPC <sessionGUID>
                     └──spawns──► job.exe JOSYN-IPC <sessionGUID>
```

`JAPServer.exe` lives for the duration of one session only. It is not a long-running daemon.
It listens on a pair of GUID-named pipes, serves the JAP protocol to the job process, and exits.

## CLI Contract

Both `JAPServer.exe` and `job.exe` receive the session GUID via their command-line arguments:

```
JAPServer.exe  JOSYN-IPC <sessionGUID>
job.exe        JOSYN-IPC <sessionGUID>
```

The GUID names the pipes both processes use to communicate with each other.
Session isolation is structural — two sessions can never interfere because each has unique pipe names.
