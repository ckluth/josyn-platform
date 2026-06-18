# ADR-024 — The Ticker: Periodic Process Launcher

**Date:** 2026-06-18
**Status:** Accepted

---

## Context

`josyn-backend` is the "scheduler and session-orchestration layer" (ADR-002). That description
covers two distinct responsibilities that, until now, were not separated clearly:

1. **Session orchestration** — deciding to start a job session and handing it off to
   `SessionLauncher`. This is the role of *orchestrators* (ADR-017B-01): `CLI`, `Listener`,
   and the as-yet-unbuilt time-based and workflow-based launchers.

2. **Periodic invocation** — waking up on a timer and deciding *which orchestrators to run*.
   Nothing in the platform fills this role yet.

ADR-017B-01 anticipated four orchestrator executables: `listener`, `ticker`, `cli`, and
`workflow-runner`. It assumed `ticker` would call `SessionLauncher` directly. That assumption
is wrong, and this ADR corrects it.

The Ticker does not know which jobs to run. That is the job of `TimeScheduler.exe` (checks
time-based job schedules) and `WorkflowRunner.exe` (checks workflow conditions). Both will call
`SessionLauncher` when their conditions are met — they are orchestrators. The Ticker only knows
*when to invoke them*. Collapsing these two concerns into a single executable would make the
Ticker responsible for scheduling logic it has no business knowing about.

Additionally, two operating modes are required from day one:

- **Windows service** — the production mode: unattended, no console, managed by the SCM.
- **Console mode** — for developer and operator use: visible output, stopped by a keypress.

This is the same dual-mode requirement that prompted the `Interactive` flag in ADR-022, but
applied at the hosting level rather than at job-process launch.

---

## Decision

### 1. Vocabulary correction

The Ticker is **not an orchestrator**. An orchestrator (ADR-017B-01) is a thin session
launcher: it constructs a `SessionStartRequest` and calls `SessionLauncher.LaunchSession()`.
The Ticker does neither.

| Term | Definition |
|---|---|
| **Orchestrator** | Thin session launcher — constructs `SessionStartRequest`, calls `SessionLauncher`. Examples: `CLI`, `Listener`, `TimeScheduler`, `WorkflowRunner`. |
| **Ticker** | Periodic process launcher — fires orchestrators on a configurable timer schedule. Has no session knowledge, no `SessionLauncher` reference, no job knowledge. |

`TimeScheduler.exe` and `WorkflowRunner.exe` are **orchestrators**, not sub-orchestrators.
The Ticker is what invokes them — not what makes them a subordinate class of thing.

### 2. The Ticker is `JOSYN.Backend.Ticker.exe` in `josyn-backend`

The Ticker is a standalone EXE produced by `josyn-backend`. It fills the `ticker-service (EXE)`
placeholder in `architecture/overview.md`. Its assembly and namespace root is
`JOSYN.Backend.Ticker`.

### 3. Two operating modes: service and console

The Ticker detects at startup whether it is running under the Windows Service Control Manager
or directly from a terminal, and behaves accordingly.

| Mode | How started | Behaviour |
|---|---|---|
| **Service** | SCM (`net start`, Task Scheduler, etc.) | No console; lifecycle managed by SCM; stops on SCM stop signal. |
| **Console** | Direct invocation from a terminal | Prints `Polling... press any key to quit.`; stops cleanly on any keypress. |

No command-line flag selects the mode. The Ticker detects service context via
`Environment.UserInteractive` (or the equivalent Windows API). The default — and the only
production mode — is service.

### 4. Fire-and-forget spawn semantics

When a target is due, the Ticker spawns it as a new process and does not wait for it to exit.
The spawned process is fully independent. The Ticker does not:

- track whether the spawned process succeeded or failed
- prevent the same target from being spawned again if it is still running when the next tick fires
- pass arguments to the spawned process

Each spawned executable is responsible for its own concurrency guard (e.g. a named mutex)
if overlapping instances would be harmful.

### 5. Schedule configuration in `josyn.bootstrap.ini`

Target definitions live in a `[Ticker-Targets]` section of `josyn.bootstrap.ini`:

```ini
[Ticker-Targets]
TimeScheduler  = TimeScheduler.exe  | 0,  30
WorkflowRunner = WorkflowRunner.exe | 15, 30
```

Format per entry:

```
<Name> = <ExeName> | <offset>, <period>
```

| Field | Meaning |
|---|---|
| `Name` | Logical name for the target (used in logs; must be unique within the section). |
| `ExeName` | The executable file name, without path. |
| `offset` | Minute-of-hour at which the first fire occurs (0–59). |
| `period` | How often to fire, in minutes (1–60). |

**Fire rule:** the target fires when `currentMinute % period == offset % period` and the
wall-clock minute has not yet triggered a fire in the current period. The Ticker evaluates
this on a one-minute internal tick.

Examples:
- `0, 30` → fires at :00 and :30 of every hour.
- `15, 30` → fires at :15 and :45 of every hour.
- `0, 60` → fires once per hour, on the hour.
- `5, 15` → fires at :05, :20, :35, :50 of every hour.

**Why `josyn.bootstrap.ini` and not the database:**
The Ticker must know what to spawn before it can establish a DB connection (or even before a
DB connection is healthy). Deployment-time target configuration belongs in bootstrap config,
not in operational data. The section is read once at startup; a restart is required to pick
up changes, which is acceptable for deployment-level configuration.

### 6. EXE path resolution by BackendRoot convention

The Ticker resolves the full path to a target executable using the BackendRoot convention
(ADR-012):

```
BackendRoot\<Name>\<ExeName>
```

where `BackendRoot` is the directory of `josyn.bootstrap.ini` and `Name` is the logical name
from the config entry.

Example deployment layout:

```
$BackendRoot\
    josyn.bootstrap.ini
    Ticker\
        JOSYN.Backend.Ticker.exe
    TimeScheduler\
        TimeScheduler.exe
    WorkflowRunner\
        WorkflowRunner.exe
    JAPServer\
        JOSYN.Jap.JAPServer.exe
    CLI\
        JOSYN.Backend.CLI.exe
```

The `<Name>` key in `[Ticker-Targets]` therefore doubles as the sub-folder name. Names must
be chosen to match the deployment directory.

### 7. `deploy-maintainer.ps1` gains a Ticker publish step

`josyn-toolbox\deploy\deploy-maintainer.ps1` gains an `Invoke-Publish` call that publishes
`JOSYN.Backend.Ticker` to `$BackendRoot\Ticker\`.

---

## Rationale

**Why not have the Ticker call `SessionLauncher` directly?**
The Ticker would then need to know which jobs to run, when, and with what arguments — exactly
the logic that belongs in `TimeScheduler.exe` and `WorkflowRunner.exe`. Separating invocation
timing from invocation content keeps each concern independently testable and deployable.

**Why INI and not the database?**
Bootstrap config is read before any other subsystem starts. The Ticker needs its target list
immediately — it cannot query a database that may itself require a healthy backend to connect.
The target list is a deployment concern (which orchestrators exist), not an operational concern
(when a specific job should run). The latter belongs to the orchestrators themselves.

**Why fire-and-forget?**
The Ticker's only responsibility is to wake up the orchestrators on time. Tracking their
outcomes would require the Ticker to understand session semantics — the exact coupling this
design avoids. If an orchestrator crashes silently, that is an observability concern for the
orchestrator and the session store, not for the Ticker.

**Why no flag for service vs. console mode?**
The mode is a property of the host environment, not of the caller's intent. A service running
under the SCM is always headless; a process launched from a terminal is always interactive.
Auto-detecting removes a footgun: an operator who forgets the flag, or a service installer
that omits it, would silently get the wrong behaviour. Detection is unambiguous and requires
no documentation.

---

## Consequences

- `josyn-backend` gains a new EXE: `JOSYN.Backend.Ticker`.
- `josyn.bootstrap.ini` gains a `[Ticker-Targets]` section. The repo copy is updated.
- `deploy-maintainer.ps1` gains a Ticker publish step.
- **`architecture/overview.md`** — the `ticker-service (EXE)` placeholder description
  ("polls for scheduled executions") is updated to reflect the correct role: periodic
  launcher of orchestrators.
- **`guides/session-launch.md`** — the Orchestrators diagram currently lists `Ticker` as a
  direct `SessionLauncher` caller. This is wrong. `Ticker` is removed from that box;
  `TimeScheduler` and `WorkflowRunner` are added in its place.
- **ADR-017B-01** — listed `ticker` as one of four orchestrator executables. That
  characterisation is corrected by this ADR: `ticker` is not an orchestrator.

---

## Open Questions

1. **Windows service registration** — how is the Ticker registered as a Windows service on a
   developer machine and in production? (installer, `sc.exe`, NSSM, or a built-in
   `IHostedService` bootstrap?) This ADR does not decide.

2. **Sub-period precision** — the one-minute internal tick is the minimum granularity. If a
   sub-minute period ever becomes necessary, the schedule format and evaluation rule would
   need extension. Not anticipated; noted for completeness.

3. **Missed-fire semantics** — if the Ticker is stopped and restarted, should it fire
   orchestrators that were missed during the downtime? Current decision: no (the Ticker fires
   only on live ticks, not on catch-up). Orchestrators that care about missed windows must
   implement their own catch-up logic.

---

## Relation to Other ADRs

- **ADR-002** (josyn-backend): the Ticker is the first scheduled-execution entry point in
  `josyn-backend`. It fills a placeholder described there.
- **ADR-012** (Maintainer Deployment): the BackendRoot convention and the `$BackendRoot\<Name>\`
  sub-folder layout govern how target EXE paths are resolved.
- **ADR-017B-01** (Session-Starter Relocation): corrected — `ticker` is not an orchestrator.
  `TimeScheduler` and `WorkflowRunner` are the orchestrators invoked by the Ticker.
- **ADR-022** (Interactive Job Launch): established the precedent for dual-mode
  (service/interactive) detection at the hosting level, applied here consistently.
