# josyn-session-broker

**Role:** Per-session boundary EXE — the only JOSYN component that simultaneously touches
both worlds of the two-worlds model. Brokers between the backend orchestration world and
the job developer's execution world via the JAP protocol over named-pipe IPC (JIP).

**Location:** `C:\DevGit\josyn-session-broker`
**Namespace:** `JOSYN.SessionBroker`
**Version:** `1.0.0-preview01`

Created per [ADR-025](../decisions/ADR-025-session-broker.md) — renamed from
`JOSYN.Jap.JAPServer` and extracted from `josyn-backend`.

---

## Decisions

| ADR | Title |
|-----|-------|
| [ADR-025](../decisions/ADR-025-session-broker.md) | SessionBroker: Rename JAPServer and Extract to Dedicated Repo |
| [ADR-018B-01](../decisions/ADR-018B-01-job-start-negotiation.md) | Job Start Negotiation (Accept / Reject) |
| [ADR-017B-03](../decisions/ADR-017B-03-credential-provider.md) | IdentityAdapter (Impersonation Extension Point) |
| [ADR-020](../decisions/ADR-020-company-adapter-model.md) | Company Adapter Model (Out-of-Process) |
| [ADR-021](../decisions/ADR-021-impersonated-process-launch.md) | Impersonated Process Launch for job.exe |
| [ADR-022](../decisions/ADR-022-interactive-job-launch.md) | Headless / Interactive Distinction for job.exe Launch |
| [ADR-004](../decisions/ADR-004-japserver-relocation.md) | JAPServer Relocation (historical — superseded by ADR-025) |

---

## Architectural position

`josyn-session-broker` is the **boundary between the two worlds**:

```
═══════════════════════════════════════════════════════════════════
  WORLD 1 — The Job Developer's World
───────────────────────────────────────────────────────────────────
  job.exe  ──(JIP named pipe, JAP protocol)──►  SessionBroker
═══════════════════════════════════════════════════════════════════
  SessionBroker  ──(SessionStore, credentials, config)──►  josyn-backend packages
═══════════════════════════════════════════════════════════════════
  WORLD 2 — The Maintainer's Backend World
```

This makes `josyn-session-broker` the structural counterpart of `josyn-job-host`:

| Repo | Side | Namespace |
|------|------|-----------|
| `josyn-job-host` | Job developer's world | `JOSYN.JobHost` |
| `josyn-session-broker` | Maintainer's world | `JOSYN.SessionBroker` |

Both use the **two-segment namespace** form — intentional and documented in ADR-025
(extending the ADR-001 rule to cover boundary-crossing EXEs).

---

## CLI contract

`SessionBroker.exe` is spawned exclusively by `SessionLauncher`. The invocation is:

```
SessionBroker.exe JOSYN-START @<path-to-session-start-spec.ini>
```

The file contains a serialised `SessionStartSpec` (via `PropertyBag`). It is read once
and deleted immediately by `Host.Entrypoint.cs`.

The `JOSYN-START` token is defined as `SessionBrokerConstants.CliModeStart` in
`JOSYN.Backend.Contracts`.

---

## Structure

```
josyn-session-broker/
├── .local-build/
│   ├── all.cmd
│   ├── build.cmd                           ← dotnet build JOSYN.SessionBroker.slnx
│   ├── build.debug.cmd
│   ├── build.release.cmd
│   ├── clean.cmd                           ← info only (EXE, no NuGet cache)
│   ├── pack.cmd                            ← info only (EXE, no NuGet package)
│   └── test.cmd                            ← info only (no test projects)
├── nuget.config                            ← points to ../local-packages/
├── JOSYN.SessionBroker.slnx
└── JOSYN.SessionBroker/
    ├── SessionBroker.cs                    ← implements IJosynApplicationProtocol (JAP handler)
    ├── Host.Entrypoint.cs                  ← startup, arg parsing, bootstrap, session flow
    ├── Host.Adapters.cs                    ← adapter EXE spawn and JIP connection
    ├── Host.Prepare.cs                     ← Turnstile, credentials, job launch, negotiation
    ├── Host.Server.cs                      ← JIP dispatch and status helpers
    ├── AdapterManager.cs                   ← holds all spawned adapter processes for the session
    ├── AdapterProcess.cs                   ← single adapter process + pipe connection
    ├── Program.cs                          ← Main → Host.Run(args)
    └── JOSYN.SessionBroker.csproj
```

### Partial class structure

`Host` is a `static partial class` split across four files by concern:

| File | Responsibility |
|------|---------------|
| `Host.Entrypoint.cs` | `Run(args)` — bootstrap, arg parsing, session flow, finalization |
| `Host.Adapters.cs` | `SpawnAdapters` — adapter EXE spawn and JIP connect loop |
| `Host.Prepare.cs` | `Prepare` — inside the Turnstile: GUID, session record, infra, credentials, job launch, negotiation |
| `Host.Server.cs` | `HandleRequest`, `HandleHandlerError`, `SetTerminalStatus` — dispatch and status helpers |

---

## Session lifecycle (runtime flow)

```
SessionLauncher.LaunchSession()
  └── spawns SessionBroker.exe JOSYN-START @<path>

Host.Run(args)
  ├── LoadBootstrapConfig()
  ├── SpawnAdapters()                        ← adapter EXEs started first; hard failure if any missing
  └── ProcessSessionStart()
        ├── ReadSessionStartSpec()           ← read + delete temp file
        ├── Turnstile.RunAsync → Prepare()
        │     ├── CreateSessionRecord()      ← GUID allocated, JobSessionRecord persisted
        │     ├── BuildJapInfrastructure()   ← SessionBroker + JipDispatcher wired
        │     ├── ResolveCredentials()       ← IdentityAdapter call via JIP
        │     ├── LaunchJobAndStorePid()     ← job.exe spawned as impersonated process
        │     └── RunNegotiation()           ← 30s timeout; AcceptSession or RejectSession
        │           ├── Accepted → NegotiationAccepted = true; Turnstile releases
        │           └── Rejected / timeout → FinishedRejected; exit 0
        └── await ServerTask                 ← serve JAP protocol until job.exe terminates
              └── Finalize terminal status
```

---

## Dependencies

All consumed via NuGet from `../local-packages/`.

```xml
<PackageReference Include="JOSYN.Foundation.JIP"                           Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Foundation.PropertyBag"                   Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Foundation.ResultPattern"                 Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Jap.Contract"                             Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Backend.Contracts"                        Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Backend.BootstrapConfig"                  Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Backend.ConfigStore"                      Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Backend.SessionStore"                     Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Backend.ErrorHandler"                     Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Adapter.IdentityAdapter.Contract"         Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Adapter.ConfigurationAdapter.Contract"    Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Commons.Log"                              Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Commons.Helpers"                          Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Commons.IdentityHelpers"                  Version="1.0.0-preview01"/>
```

Runtime spawn relationships (not NuGet dependencies):
- Spawned by `SessionLauncher` (in `josyn-backend`) — via `SessionBrokerConstants`
- Spawns adapter EXEs (e.g. `Contoso.IdentityAdapter.exe`) via `JOSYN-ADAPTER <guid>`
- Spawns `job.exe` (from job repository) as an impersonated process via `JOSYN-IPC <guid>`

---

## Build & Package

```
.local-build\build.cmd [Release|Debug]    ← builds the solution (default: Release)
.local-build\build.debug.cmd             ← shorthand for Debug build
.local-build\build.release.cmd           ← shorthand for Release build
```

This is an EXE — no `pack.cmd` (no NuGet package produced).
NuGet feed: `../local-packages/` (sibling directory).

License: MIT | Company: HAEVG AG | Target: net10.0

---

## Exit codes

| Code | Meaning |
|------|---------|
| `0` | Session completed successfully (or definitively rejected — normal outcome) |
| `1` | Fatal error (bootstrap failure, IPC error, spawn failure, unhandled exception) |

---

## Sanity Notes

### Structural specifics

- **Console EXE** — `<OutputType>Exe</OutputType>`. Must NOT have `<GenerateDocumentationFile>`
  or NuGet packaging metadata (`<PackageId>`, `<PackageTags>`, etc.).
- **Two-segment namespace** `JOSYN.SessionBroker` — intentional per ADR-025 (extends ADR-001).
  Not a naming violation.
- **`SessionBroker` is `internal sealed class`** — correct; it has instance state implementing
  `IJosynApplicationProtocol`. Static is not applicable here.
- **`Host` is `internal static partial class`** — correct; split across four files by concern.
- **`Program` is `internal class`** (not static) — correct for a top-level entry point.
- **No test project** — known gap, not a violation.

### Dependency constraints

- All `josyn-backend` packages are consumed via NuGet only. No project references cross the
  repo boundary — this is the structural rule enforced by ADR-004 (governance note) and
  reaffirmed by ADR-025.
- `JOSYN.Adapter.IdentityAdapter.Contract` and `JOSYN.Adapter.ConfigurationAdapter.Contract`
  come from `josyn-adapter-contracts` (a separate repo) — NuGet, not project references.
- This repo must never reference `josyn-job-host` packages — `job.exe` is a runtime peer,
  not a library dependency.
- This repo must never be referenced as a NuGet dependency by any other repo — it is an EXE,
  not a library.
