# ADR-007B-01 — Backend Restructuring: Trigger Executables, ErrorHandler, and SessionStarter Fire-and-Forget Correction

**Date:** 2026-06-04
**Status:** Proposed

---

## Context

The current `josyn-backend` repo contains five building blocks:

| Component | Type | Role |
|-----------|------|------|
| `JOSYN.Backend.GlobalConfig` | NuGet (Cat A) | Runtime config contract + PoC placeholder |
| `JOSYN.Backend.SessionStore` | NuGet (Cat A) | Session persistence via EF Core |
| `JOSYN.Backend.SessionStarter` | NuGet (Cat A) | Allocates GUID, persists session, spawns JAPServer |
| `JOSYN.Jap.JAPServer` | EXE (Cat B) | Per-session JAP protocol server |
| `JOSYN.Backend.Demo.FakeSessionStarterConsumer` | EXE (Cat B, PoC demo) | End-to-end round-trip demonstration |

This structure has three gaps that must be addressed before real execution logic is added:

**Gap 1 — No trigger executables.**
The backend has `SessionStarter` but nothing that calls it in production. The legacy system has three distinct trigger roles: a remote-request server (`JobSystem.Service`, WCF), a timer poller (`JobSystem.TriggerAgent`), and a manual operator CLI (also part of `TriggerAgent`). JOSYN must have structural equivalents — even as stubs — before any trigger logic is written, to ensure that logic lands in the right place.

**Gap 2 — No error reporting endpoint.**
`LocalLog` (from `josyn-jap`) is a process-local file logger used by JAPServer and job executables. It was never intended as the platform's error reporting authority. There is currently no component that handles error notification (alerting an operator) or durable error storage (beyond the local log file). Every component that encounters an unrecoverable error has nowhere to send it. This gap must be closed with a stable contract before more components are added.

**Gap 3 — SessionStarter's `StartSession` contains a faulty spawn-site probe.**
The current implementation issues a 500 ms `WaitForExit` call immediately after `Process.Start`. The intent was to detect an immediate JAPServer crash and surface it at the caller. The mechanism is unreliable: a JAPServer that crashes 501 ms after start passes the probe and returns false success; a JAPServer that starts but deadlocks on pipe creation also passes. More importantly, the probe *delays every healthy call by 500 ms* — an overhead that compounds across concurrent sessions. The correct mechanism for detecting a crashed JAPServer is a session status column in the store; that is a documented PoC limitation addressed in a later session. The probe must be removed to make `StartSession` a true fire-and-forget call.

---

## Decision

### 1. Three new trigger executables: Listener, Ticker, CLI

Three new Category B executables are introduced as structural stubs:

| Executable | Replaces | Role |
|------------|----------|------|
| `JOSYN.Backend.Listener` | `JobSystem.Service` (WCF) | Future REST server; receives remote "start session" requests |
| `JOSYN.Backend.Ticker` | `JobSystem.TriggerAgent` (timer role) | Future Windows Service; polls once per minute for scheduled sessions and workflow-triggered sessions |
| `JOSYN.Backend.CLI` | `JobSystem.TriggerAgent` (CLI role) | Manual session start; maintenance, integration testing, demonstration |

All three remain totally empty stubs in this iteration. Their purpose is structural: to establish the correct solution membership, dependency graph, and classification before any trigger logic is written.

### 2. Three separate solutions — one per trigger executable

Each trigger executable lives in its own Pattern B sub-folder and solution:

```
josyn-backend-listener/
├── nuget.config
├── JOSYN.Backend.Listener.slnx
├── .local-build/
│   ├── build.cmd
│   └── clean.cmd
└── JOSYN.Backend.Listener/
    └── Program.cs                  ← EXE stub; references SessionStarter via NuGet

josyn-backend-ticker/
├── nuget.config
├── JOSYN.Backend.Ticker.slnx
├── .local-build/
│   ├── build.cmd
│   └── clean.cmd
└── JOSYN.Backend.Ticker/
    └── Program.cs                  ← EXE stub; references SessionStarter via NuGet

josyn-backend-cli/
├── nuget.config
├── JOSYN.Backend.CLI.slnx
├── .local-build/
│   ├── build.cmd
│   └── clean.cmd
└── JOSYN.Backend.CLI/
    └── Program.cs                  ← EXE stub; references SessionStarter via NuGet
```

Each solution is self-contained. `SessionStarter` is consumed via `PackageReference` from `local-packages`, not via a cross-solution project reference.

Listener, Ticker, and CLI will evolve into components with completely different runtime profiles (REST server, Windows Service, interactive tool). Sharing a solution would couple their release cycles, build configurations, and dependency sets before those profiles are known. Establishing separate solutions now, when the work is cheapest, prevents a future session from unravelling solution structure.

### 3. New NuGet: JOSYN.Backend.ErrorHandler

`JOSYN.Backend.ErrorHandler` is a new Category A NuGet and the platform's single error-reporting authority. It lives in its own solution:

```
josyn-backend-error-handler/
├── nuget.config
├── JOSYN.Backend.ErrorHandler.slnx
├── .local-build/
│   ├── build.cmd
│   ├── clean.cmd
│   └── pack.cmd
└── JOSYN.Backend.ErrorHandler/
    ├── Contracts/
    │   └── IErrorHandler.cs
    └── ErrorHandler.cs                 ← first version: file-system log + Console.Write
```

First-version responsibilities: receive an error report, write it to a file-system log, and write to the console. Notification and durable storage are deferred.

### 4. SessionStarter fire-and-forget correction

`SessionStarter.StartSession` is corrected: after `Process.Start`, the method returns `sessionGuid` immediately without waiting or probing the JAPServer process. The 500 ms `WaitForExit` probe is removed. Orphaned sessions resulting from a JAPServer crash are the responsibility of a future session-status mechanism — this is an already-documented PoC limitation and the correct fix is a status column in the session store, not a fragile spawn-site probe.

### 5. JAPServer gains ErrorHandler dependency

`JOSYN.Jap.JAPServer` adds `JOSYN.Backend.ErrorHandler` as a NuGet dependency. This is the only change to the existing JAPServer.

### 6. What does not change

- `JOSYN.Backend.GlobalConfig` — unchanged
- `JOSYN.Backend.SessionStore` — unchanged (the `SessionStore → GlobalConfig` dependency, noted in the prompt, is deferred to a later session)
- `JOSYN.Backend.SessionStarter` — NuGet classification and solution placement unchanged; only the 500 ms probe is removed
- `JOSYN.Jap.JAPServer` — logic unchanged; dependency list gains `ErrorHandler` only
- `JOSYN.Backend.Demo.FakeSessionStarterConsumer` — out of scope for this restructuring; addressed in a dedicated future session

---

## Resulting structure

```
josyn-backend/
├── .local-build/
│   ├── build.cmd         ← updated: adds listener, ticker, cli, error-handler
│   └── pack.cmd          ← updated: adds error-handler
├── local-packages/
├── josyn-backend-global-config/          (unchanged)
├── josyn-backend-session-store/          (unchanged)
├── josyn-backend-session-starter/        (500ms probe corrected; NuGet classification unchanged)
├── josyn-backend-error-handler/          ← NEW
├── josyn-backend-listener/               ← NEW
├── josyn-backend-ticker/                 ← NEW
├── josyn-backend-cli/                    ← NEW
├── josyn-backend-jap-server/             (ErrorHandler ref added)
└── josyn-backend-demo/                   (out of scope)
```

---

## Dependency graph (complete, post-restructuring)

```
JOSYN.Foundation.ResultPattern   (no deps)
JOSYN.Backend.GlobalConfig       (no deps)
JOSYN.Backend.SessionStore       → ResultPattern
JOSYN.Backend.ErrorHandler       → ResultPattern, GlobalConfig

JOSYN.Backend.SessionStarter     → ResultPattern, SessionStore, GlobalConfig    [NuGet, Cat A]
JOSYN.Backend.Listener           → SessionStarter, GlobalConfig, ErrorHandler, ResultPattern
JOSYN.Backend.Ticker             → SessionStarter, GlobalConfig, ErrorHandler, ResultPattern
JOSYN.Backend.CLI                → SessionStarter, GlobalConfig, ErrorHandler, ResultPattern

JOSYN.Jap.JAPServer              → SessionStore, GlobalConfig, ErrorHandler,
                                    JIP, PropertyBag, ResultPattern,
                                    Jap.Shared.Contract, Jap.Shared.Log
```

---

## Consequences

- The trigger layer has named structural homes before any trigger logic is written; future implementations land in the correct executable without architectural debate.
- Each trigger executable occupies its own solution; it can evolve, be deployed, and be released independently without affecting the others.
- `SessionStarter`'s NuGet classification is reinforced: a stable shared component consumed by three independent solutions via `PackageReference` is exactly the Category A model.
- `ErrorHandler` establishes the contract for error reporting across the entire backend before multiple consumers invent independent mechanisms.
- The solution-boundary rule is preserved: no cross-solution project references exist; `SessionStarter` is consumed via `PackageReference` by each trigger solution.
- The 500 ms probe is removed; the PoC limitation it was obscuring is now visible and documented, awaiting the correct fix (session status column).
- `JOSYN.Backend.Demo.FakeSessionStarterConsumer` currently takes `SessionStarter` as a NuGet reference. It is unaffected by this restructuring, since `SessionStarter`'s NuGet identity does not change. The demo remains deferred to a dedicated future session.

---

## Decision challenge — objections and rebuttals

### Challenge 1 — "SessionStarter should be reclassified to ClassLibrary — its only consumers are in this repo"

*`SessionStarter` is never consumed outside `josyn-backend`. Its three consumers — Listener,
Ticker, CLI — are sibling solutions in the same repo. A ClassLibrary is simpler: no versioning,
no pack step, no restore cycle. Packaging it as a NuGet for intra-repo use is unnecessary
ceremony.*

**Rebuttal:** The Category A / Category B model is defined by solution boundaries, not repo
boundaries. Cross-solution project references are forbidden by the platform's structural rule.
`SessionStarter` is consumed by three separate solutions; the only mechanism for sharing a
compiled artifact across solutions without violating that rule is a NuGet package from
`local-packages`. The "intra-repo" qualifier does not change the structural constraint. Furthermore,
`SessionStarter` is explicitly described as stable, low-churn shared infrastructure — exactly the
profile for which the packaging overhead is lowest. A component that seldom changes has low
versioning cost and high sharing benefit; the overhead argument weighs least against the component
least likely to generate it.

---

### Challenge 2 — "Three separate solutions for three nearly identical EXE stubs is over-engineering — one solution would be simpler"

*Listener, Ticker, and CLI are currently empty stubs. They are nearly identical: one `Program.cs`,
one set of NuGet references, one `.local-build`. Maintaining three separate solutions, build
scripts, and `nuget.config` files for three stub projects imposes structural overhead
with no immediate benefit.*

**Rebuttal:** The overhead at stub stage is exactly one `Program.cs` and a build script per
project. The cost of separation now is trivial; the cost of separation later — after Listener
is a REST server with middleware, Ticker is a Windows Service with a scheduler, and CLI is an
interactive tool with argument parsing — is a session of restructuring work plus ADR updates.
Pattern B's unit is the independently releasable deliverable. Listener, Ticker, and CLI are
three independent deliverables: they have different runtime hosts, different deployment targets,
different release cadences. Grouping them in one solution because they are currently empty makes
the solution structure describe the current (temporary) state rather than the (permanent)
architectural intent. The platform's structural-commitment-first approach: decide structure when
the slate is clean, not when it is too late.

---

### Challenge 3 — "ErrorHandler as a NuGet is premature — the first version is just a file write"

*Introducing a versioned NuGet artifact for a component whose initial implementation is
`Console.Write` and a file append adds overhead without substance. It can be added when
there is real logic to ship.*

**Rebuttal:** The contract, not the implementation, is what needs to exist now. Every
executable and JAPServer needs an error reporting destination. Without `IErrorHandler`,
each component will invent its own mechanism — the same antipattern that `IGlobalConfig`
was introduced to prevent (ADR-006B-02 Context). The minimal first implementation is
intentional: it establishes the seam while adding no risk. Replacing `FileSystemErrorHandler`
with a real notification and storage implementation later requires only a change at the
composition root. Deferring the NuGet means that when real error handling logic is finally
needed, all consumers must be retrofitted simultaneously — expensive and error-prone.

---

### Challenge 4 — "Removing the 500 ms probe makes JAPServer startup failures silent"

*The probe is imperfect, but it is the only immediate feedback path. Removing it means
`StartSession` returns `Ok(guid)` even when JAPServer never starts. The caller has no way
to distinguish a healthy session from a dead one.*

**Rebuttal:** The probe's guarantee is illusory. A JAPServer that crashes 501 ms after
start, or one that starts but immediately deadlocks on pipe creation, passes the probe and
returns false success. More importantly, the probe *delays every healthy call by 500 ms* —
an overhead that compounds across concurrent sessions. The correct mechanism is already
identified: a session status column in the store. `SessionStarter` writes `status=Pending`;
JAPServer writes `status=Running` on startup and `status=Completed` or `status=Failed` on
exit. The absence of a status transition within a configurable timeout is the correct crash
signal — robust, auditable, and handled outside the spawn site by the component responsible
for monitoring (the future `Ticker` or a watchdog). Removing the probe does not create a
new failure mode; it makes an existing undocumented failure mode explicit and directs it to
its correct future owner.

---

### Challenge 5 — "CLI has no defined behavior — adding it now is speculation dressed up as architecture"

*Listener and Ticker have future-state descriptions (REST server, Windows Service timer).
CLI is defined only as "manual session start." Adding an empty executable with no concrete
specification is premature commitment to a name and a home.*

**Rebuttal:** "Manual session start" is a sufficiently concrete role. It replaces the CLI
functionality that `JobSystem.TriggerAgent` had — a real, exercised capability in the legacy
system. The legacy precedent is documentation enough. Adding it now, while the trigger solutions
are being created, costs nothing: one project file, one `Program.cs` with a `// TODO` comment,
one entry in the build script. Deferring it means a future session must revisit the solution
structure, the build scripts, the ADR, and this discussion — all to add what is already known
to be needed. The rule that governs: structural decisions are cheapest when the slate is clean.
The CLI's home is decided here; its behavior is not.
