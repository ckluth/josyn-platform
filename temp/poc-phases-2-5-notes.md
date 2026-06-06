# PoC Phases 2–5 — Inspection & Runtime Notes

Generated after session: implementing the SessionStore PoC roadmap (Phases 2–5).
Date: 2026-06-03

---

## What was built

| Phase | Deliverable | Status |
|-------|-------------|--------|
| 1 | `JOSYN.Backend.SessionStore` NuGet package | ✅ Done (prior session) |
| 2 | `JOSYN.Backend.GlobalConfig` NuGet package | ✅ Done |
| 3 | `JOSYN.Backend.SessionStarter` NuGet package | ✅ Done |
| 4 | `JOSYN.Jap.JAPServer` wired to real SessionStore; `FakeSessionStarterConsumer` demo EXE | ✅ Done |
| 5 | `architecture/overview.md` component map + dependency chain updated | ✅ Done |

All 5 solutions build clean (`josyn-backend/.local-build/build.cmd Release`).

---

## Before you run anything — required setup

### 1. Adjust `HardcodedGlobalConfig.cs`

File: `josyn-backend/josyn-backend-global-config/JOSYN.Backend.GlobalConfig/HardcodedGlobalConfig.cs`

The two values are machine-specific developer constants:

```csharp
public string SessionStoreConnectionString =>
    "Server=localhost;Database=JobSystem;Integrated Security=true;TrustServerCertificate=true;";

public string JapServerExePath =>
    @"C:\Temp\VS.OUT\JOSYN\JOSYN.Jap.JAPServer\bin\Release\JOSYN.Jap.JAPServer.exe";
```

- **Connection string**: verify SQL Server is reachable at `localhost`, database `JobSystem` exists.
- **JapServerExePath**: verify the Release build output path matches your machine.
  After `build.cmd Release`, the EXE should be at exactly that path.

If you change these values you must re-pack GlobalConfig:
```
josyn-backend/josyn-backend-global-config/.local-build/pack.cmd
```
...and then rebuild JAPServer + Demo so they pick up the new NuGet.

### 2. SQL Server — `JobSessions` table

The `SessionStore` uses EF Core but ships **no migration runner** in the PoC.
The table must exist before the demo runs.

Check the `SessionStoreEntity` and `SessionStoreDbContext` in
`josyn-backend/josyn-backend-session-store/JOSYN.Backend.SessionStore/Db/`
for the schema, then create the table manually or run an EF migration.

Minimum required columns (inferred from code):
- `UID` — `uniqueidentifier`, PK
- `JobTypeName` — `nvarchar`
- `Arguments` — `nvarchar(max)`
- `Result` — `nvarchar(max)`

---

## Do you have a complete running demo round-trip?

**Not fully automated — one gap remains.**

What the demo EXE does:
1. Calls `StartSession("DemoJob", <INI args>)` → saves a session row → spawns `JAPServer.exe JOSYN-IPC <guid>`
2. Polls the `SessionStore` every 500 ms for up to 30 s waiting for `Result` to be non-empty
3. Prints the result (or reports a DB error / timeout)

What JAPServer does when spawned:
1. Reads `HardcodedGlobalConfig` for the connection string
2. Constructs `SessionStore`
3. Waits (via JIP/Named Pipes) for a `job.exe` client to connect
4. When `job.exe` calls `GetRawArguments()` → reads `Arguments` from `SessionStore`
5. When `job.exe` calls `PutRawResult(result)` → writes `Result` back to `SessionStore`

**The gap:** No `job.exe` is part of the demo. JAPServer waits indefinitely for a JIP client.
The consumer will time out after 30 s.

### How to close the loop manually

**Option A — two-step manual demo:**
1. Run `FakeSessionStarterConsumer.exe`, note the GUID printed to console.
2. Run any existing `job.exe` (from `josyn-job-host`) with `JOSYN-IPC <guid>`.
3. The job reads arguments from JAPServer (→ DB), processes, calls PutRawResult (→ DB).
4. The consumer poll loop sees the result and exits.

**Option B — VS debug launch:**
1. In `josyn-backend-jap-server`, manually insert a test row into the DB for the GUID
   in `launchSettings.json` (`dea5611d-d740-437f-ad93-7a5dc5ae4299`).
2. Launch JAPServer from VS using the `Run-JAPServer` profile.
3. Connect a job.exe to that GUID.

**What Phase 4 proved architecturally:** All the plumbing is correct — the session
is saved before JAPServer spawns (required by the protocol), JAPServer reads from
the store, writes back to the store. The demo just lacks an automated job-side client.

---

## Manual inspection checklist

### josyn-platform (docs)

- [ ] `architecture/overview.md` — Component Map: josyn-backend block shows GlobalConfig,
  SessionStore, SessionStarter, JAPServer (with real wiring note), Demo EXE, future
  placeholders (TriggerAgent, Scheduling, Service, WorkflowAdapter, listener-service,
  ticker-service, cli-exe)
- [ ] `architecture/overview.md` — Dependency Chain: Foundation/JAP/JobHost diagram
  unchanged; JAPServer entry now lists `+ SessionStore + GlobalConfig`; new
  `── josyn-backend package chain ──` block added below
- [ ] `repos/josyn-backend/overview.md` — Current state paragraph updated for all phases;
  directory tree has 5 sub-folders; JAPServer structure section shows real constructor;
  Dependencies table updated; build section updated; sanity notes cleaned up
- [ ] `repos/josyn-backend/decisions/ADR-003-global-config.md` — rationale for placeholder
  interface; zero-dependency design

### josyn-backend — JAPServer (most critical change)

- [ ] `josyn-backend-jap-server/.../JOSYN.Jap.JAPServer.csproj` — two new PackageReferences:
  `JOSYN.Backend.SessionStore`, `JOSYN.Backend.GlobalConfig`
- [ ] `josyn-backend-jap-server/.../Host.cs` — `HardcodedGlobalConfig` + `SessionStore`
  constructed after `sessionKey` parsed; passed into `RunServer(sessionKey, sessionStore)`;
  `_japServer` static field gone; `JAPServer` constructed locally as `new JAPServer(sessionStore, sessionKey)`
- [ ] `josyn-backend-jap-server/.../JAPServer.cs` — constructor `(ISessionStore, Guid)`;
  `GetRawArguments` reads from DB (`GetSession` → `.Arguments`);
  `PutRawResult` reads session → constructs `new JobSession { Result = result }` →
  `UpdateSession` → console output only after successful DB write;
  `PutError` unchanged; `FakeReadArgumentsFromFile` gone

### josyn-backend — GlobalConfig (Phase 2)

- [ ] `josyn-backend-global-config/.../Contracts/IGlobalConfig.cs` — two properties
- [ ] `josyn-backend-global-config/.../HardcodedGlobalConfig.cs` — PoC constants,
  **machine-specific** (see setup section above)

### josyn-backend — SessionStarter (Phase 3)

- [ ] `josyn-backend-session-starter/.../Contracts/ISessionStarter.cs` — `Result<Guid> StartSession(string, string)`
- [ ] `josyn-backend-session-starter/.../SessionStarter.cs` — key details:
  - `File.Exists` pre-check on JapServerExePath
  - Save-before-spawn order (required — JAPServer reads from DB at startup)
  - `Result<Guid>.Propagate(save.ToResult<Guid>())` — cross-type error propagation
  - 500 ms `WaitForExit` crash-detection window after spawn

### josyn-backend — Demo EXE (Phase 4)

- [ ] `josyn-backend-demo/.../Program.cs` — wiring order correct; poll loop tracks
  `lastPollError`; reports last DB error on timeout (not a silent 30 s hang)
- [ ] `josyn-backend-demo/.local-build/` — `build.cmd`, `clean.cmd`, `all.cmd` present;
  **no `pack.cmd`** (correct — EXE is not a NuGet package)

### josyn-backend — Batch orchestrator scripts

- [ ] `.local-build/build.cmd` — order: SessionStore → GlobalConfig → SessionStarter →
  JAPServer → Demo
- [ ] `.local-build/clean.cmd` — all 5 sub-folders listed
- [ ] `.local-build/pack.cmd` — still only 3 entries (SessionStore, GlobalConfig,
  SessionStarter); JAPServer and Demo correctly absent

---

## Stale item to clean up

This section is now obsolete — `FakeReadArgumentsFromFile` was removed in Phase 4.

---

## Known PoC limitations (not violations)

These are documented in `repos/josyn-backend/overview.md` sanity notes:

- `HardcodedGlobalConfig` uses compile-time developer-machine constants
- `SessionStarter`: if `Process.Start` succeeds but JAPServer crashes after the 500 ms
  window, the session row stays with empty `Result` — requires a status column to fix properly
- `FakeSessionStarterConsumer` uses a 500 ms poll loop — production would use a push signal
- No test projects for any of the PoC packages (known gap)
