# josyn-jrp

**Role:** JRP protocol contracts — wire-safe request/response records for the JOSYN REST Protocol
that bridges the surface human window to the platform-resident Gateway (ADR-033).

**Location:** `C:\DevGit\josyn-jrp`
**Version:** `1.0.0-preview01`

---

## Structure

```
josyn-jrp/
├── josyn-jrp-launch/             ← NuGet library — session-launch contracts
│   └── JOSYN.Jrp.Launch/
└── josyn-jrp-surface/            ← NuGet library — surface read/write contracts
    └── JOSYN.Jrp.Surface/
```

Pattern B repo: two sub-solutions, each with its own `.local-build/` (clean, build, pack).
The root `.local-build/pack.cmd` orchestrates both in dependency order (Launch → Surface).

---

## JOSYN.Jrp.Launch

**Purpose:** Shared identity and error vocabulary for all JRP requests. A thin client that
binds only to launch operations (e.g. `SessionClient`) takes only this package.

**Dependencies:** `JOSYN.Jap.Contract`

### Key types

| Type | Role |
|------|------|
| `JrpTarget` | Identifies the machine and environment a request is addressed to (`RuntimeEnvironment` + `Machine`). Carried by every JRP request from the first version (ADR-031 DS-2). |
| `JrpErrorCategory` | Stable error taxonomy: `NotFound`, `Invalid`, `Unauthorized`, `Unavailable`, `Timeout`, `Internal`. Shared by both contract families. |
| `StartSessionRequest` | Request record to start a named job session. |
| `StartSessionResponse` | Response record carrying the allocated session GUID. |

---

## JOSYN.Jrp.Surface

**Purpose:** Read queries and control commands for the JRP surface concern — the types that
cross the `ISurfaceAgent` seam between the surface CLI/Blazor and the platform-resident
`GatewayCommandHandler`.

**Dependencies:** `JOSYN.Jrp.Launch`, `JOSYN.Foundation.ResultPattern`, `JOSYN.Backend.Contracts`

### Key types

| Type | Namespace | Role |
|------|-----------|------|
| `JrpError` | `JOSYN.Jrp.Surface` | Produces categorised `Error` values; encodes `JrpErrorCategory` as a `[Category]` prefix on the message for round-trip recovery. |
| `ChangeJobArgument` | `JOSYN.Jrp.Surface` | Write command: change an existing argument record. Full envelope: actor, target, correlation ID, job name, argument name, content. |
| `ArgumentChangeOutcome` | `JOSYN.Jrp.Surface` | Write response DTO: job name, argument name, before/after content. |
| `GetRecentSessions` | `JOSYN.Jrp.Surface.Queries` | Query: most recent sessions on a target, bounded by `MaxCount`. |
| `GetErrorDetail` | `JOSYN.Jrp.Surface.Queries` | Query: full detail of a single error by UID. |
| `GetRegisteredJobs` | `JOSYN.Jrp.Surface.Queries` | Query: all registered jobs as a lightweight listing. |
| `GetJobArguments` | `JOSYN.Jrp.Surface.Queries` | Query: all argument records for a single registered job. |
| `GetJobSchedule` | `JOSYN.Jrp.Surface.Queries` | Query: schedule (and entries) for a single registered job. |
| `SessionSummary` | `JOSYN.Jrp.Surface` | DTO: session identity, status, timing, user, machine, environment. |
| `ErrorDetail` | `JOSYN.Jrp.Surface` | DTO: error UID, causer, message, call stack, exception details. |
| `RegisteredJobSummary` | `JOSYN.Jrp.Surface` | DTO: job name, technical user, argument count. |
| `JobArguments` | `JOSYN.Jrp.Surface` | DTO: job registration with ordered argument records. |
| `ArgumentSummary` | `JOSYN.Jrp.Surface` | DTO: single argument record name and content. |
| `JobSchedule` | `JOSYN.Jrp.Surface` | DTO: schedule header (suspended flag, suspend-until date) and entries. |
| `ScheduleEntrySummary` | `JOSYN.Jrp.Surface` | DTO: schedule entry (argument record name, cron definition, tolerance). |

**Design constraints (ADR-031, ADR-033):**
- No DB shape appears here — DTOs are transport-safe and have no EF Core dependency.
- `JrpErrorCategory` lives in `JOSYN.Jrp.Launch` so both contract families share one vocabulary.

---

## Build & Package

```
josyn-jrp/.local-build/clean.cmd    ← clears user NuGet cache for both packages
josyn-jrp/.local-build/pack.cmd     ← packs Launch then Surface to ../../local-packages/
```

License: MIT | Company: HAEVG AG | Target: net10.0

---

## Sanity Notes

### Structural specifics
- **Pattern B repo**: two sub-solutions in kebab sub-folders. See `architecture/repo-structure-conventions.md`.
- No test project — contracts are pure records, mirroring `josyn-jap`'s `JOSYN.Jap.Contract`. Behavioural coverage lives in `josyn-surface`'s test project.

### Dependency constraints
- `JOSYN.Jrp.Launch` may only reference `JOSYN.Jap.Contract`. Any reference to backend packages or EF Core is a violation.
- `JOSYN.Jrp.Surface` may reference `JOSYN.Jrp.Launch`, `JOSYN.Foundation.ResultPattern`, and `JOSYN.Backend.Contracts` (for `ExecutionStatus`). No DB shape, no EF Core.

### Known exceptions
- No EXE project — this repo is contracts only.
