> **Correction (ADR-032, 2026-06-23):** `Listener` is removed from the platform's orchestrator
> set. No standalone `JOSYN.Backend.Listener` EXE is built. Network-initiated session launch
> (`start-session`) is a verb on the surface agent's REST API (ADR-030 D-16/D-19). See ADR-032.

# ADR-024 — The Ticker: Periodic Process Launcher

**Date:** 2026-06-18
**Status:** Accepted

---

## Context

`josyn-backend` is the "scheduler and session-orchestration layer" (ADR-002). That description
covers two distinct responsibilities that, until now, were not separated clearly:

1. **Session orchestration** — deciding to start a job session and handing it off to
   `SessionLauncher`. This is the role of *orchestrators* (ADR-017B-01): `CLI`,
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
| **Orchestrator** | Thin session launcher — constructs `SessionStartRequest`, calls `SessionLauncher`. Examples: `CLI`, `TimeScheduler`, `WorkflowRunner`. |
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
| **Console** | Direct invocation from a terminal | Prints status lines every tick; stops cleanly on any keypress. |

No command-line flag selects the mode. The Ticker detects service context via
`Environment.UserInteractive` (or the equivalent Windows API). The default — and the only
production mode — is service.

The mode also governs how target processes are spawned:

| Mode | Target spawn | Console output |
|---|---|---|
| **Service** | `UseShellExecute = false`, `CreateNoWindow = true` — headless | Silent |
| **Console** | `UseShellExecute = true` — each target opens in its own console window | Per-tick status line; one action line per due target |

Console-mode output format per tick:
```
[HH:mm:ss]  running: TimeScheduler (PID 12345)
  WorkflowRunner       fired    — PID 12400
  TimeScheduler        skipped  — PID 12345 still running
```
The header line lists all targets currently alive. Action lines appear only when a target is due.

### 4. Single-instance guard — skip if previous run is still active

When a target is due, the Ticker checks whether the previous instance of that target is still
running before spawning a new one.

- **Previous instance exited**: dispose the process handle and spawn a new instance.
- **Previous instance still running**: skip this tick for that target. Log a "skipped" line in
  console mode. No second instance is started.

The Ticker does not:
- wait for the spawned process to exit
- pass arguments to the spawned process
- track exit codes or error output from spawned processes

The Ticker retains a `Process` handle per target solely to read `HasExited`. It disposes the
handle as soon as the process is confirmed exited and a new instance is about to start.

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
| `offset` | Second within a minute at which the first fire occurs (0–59). |
| `period` | How often to fire, in seconds (1–60). |

**Fire rule:** the target fires when `currentSecond % period == offset % period`. The Ticker
evaluates this on a one-second internal tick. Prevention of double-firing within the same
period is handled by the single-instance guard (§4): a just-spawned process will not have
exited yet, so the next tick — even if it satisfies the formula — will be skipped.

Examples:
- `0, 30` → fires at second :00 and :30 of every minute.
- `15, 30` → fires at second :15 and :45 of every minute.
- `0, 60` → fires once per minute, on the minute.
- `5, 15` → fires at second :05, :20, :35, :50 of every minute.

**Why `josyn.bootstrap.ini` and not the database:**
The Ticker must know what to spawn before it can establish a DB connection (or even before a
DB connection is healthy). Deployment-time target configuration belongs in bootstrap config,
not in operational data. The section is read once at startup; a restart is required to pick
up changes, which is acceptable for deployment-level configuration.

### 6. EXE path resolution — `Orchestrators\<Name>\<ExeName>`

The Ticker resolves the full path to a target executable under the `Orchestrators` sub-folder
of `BackendRoot` (ADR-012):

```
BackendRoot\Orchestrators\<Name>\<ExeName>
```

where `BackendRoot` is the directory of `josyn.bootstrap.ini` and `Name` is the logical name
from the config entry.

Although the Ticker is not itself an orchestrator, it lives alongside the orchestrators it
manages. All scheduler-related executables share the `Orchestrators\` folder for a cleaner
deployment layout.

Example deployment layout:

```
$BackendRoot\
    josyn.bootstrap.ini
    Orchestrators\
        Ticker\
            JOSYN.Backend.Ticker.exe
        TimeScheduler\
            TimeScheduler.exe
        WorkflowRunner\
            WorkflowRunner.exe
        CLI\
            JOSYN.Backend.CLI.exe
    JAPServer\
        JOSYN.Jap.JAPServer.exe
    Adapters\
        IdentityAdapter\
            Contoso.IdentityAdapter.exe
        ConfigurationAdapter\
            Contoso.ConfigurationAdapter.exe
    JobRepository\
        <JobName>\
            <job>.exe
```

The `<Name>` key in `[Ticker-Targets]` doubles as the sub-folder name under `Orchestrators\`.
Names must be chosen to match the deployment directory.

### 7. `deploy-maintainer.ps1` gains a Ticker publish step

`josyn-toolbox\deploy\deploy-maintainer.ps1` gains an `Invoke-Publish` call that publishes
`JOSYN.Backend.Ticker` to `$BackendRoot\Orchestrators\Ticker\`.

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

**Why not pure fire-and-forget?**
A pure fire-and-forget model is simple but produces a footgun: if an orchestrator run takes
longer than its period (e.g., a TimeScheduler that queries a slow database or waits on a hung
job), the next tick would spawn a second instance, which would query the same schedules and
potentially launch duplicate job sessions. Tracking the PID and guarding against overlap
eliminates this risk at the Ticker layer. Targets do not need to implement their own concurrency
guard for overlapping-tick protection — the Ticker will not spawn a second instance until the
first has exited. Exit-code and error-output tracking is deliberately excluded: the Ticker's
responsibility ends at spawn-time.

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

2. **Sub-second precision** — the one-second internal tick is the minimum granularity. If a
   sub-second period ever becomes necessary, the schedule format and evaluation rule would
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
