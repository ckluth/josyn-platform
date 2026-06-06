# SessionStore -> josyn-backend PoC Roadmap

## Starting point

- `josyn-sandbox/.../SessionStore` - working prototype; contracts, EF Core impl, DB entity all present
- `ADR-001` mandates: Category A NuGet + separate `.Mock` companion, one solution per building block
- `JOSYN.Jap.JAPServer` is the first real consumer - currently uses hardcoded fake args/result

---

## Phase 1 - `JOSYN.Backend.SessionStore` NuGet package

New Pattern-B sub-folder: `josyn-backend/josyn-backend-session-store/`

**`JOSYN.Backend.SessionStore`** (NuGet library)
- `Contracts/IJobSession.cs`, `Contracts/ISessionStore.cs`
- `JobSession.cs` - sealed record
- `SessionStore.cs` - EF Core production implementation; accepts `string connectionString` via constructor
- `Db/SessionStoreDbContext.cs` (internal), `Db/SessionStoreEntity.cs` (internal)
- Dependencies: `JOSYN.Foundation.ResultPattern`, `Microsoft.EntityFrameworkCore.SqlServer`

Supporting: solution, `nuget.config`, `.local-build/`

Documentation:
- `ADR-002-session-store.md` in `josyn-platform/repos/josyn-backend/decisions/`
- Update `josyn-platform/repos/josyn-backend/overview.md`
- Add to `josyn-backend/.local-build/build.cmd` and `pack.cmd`

---

## Phase 2 - `JOSYN.Backend.GlobalConfig` NuGet package

New Pattern-B sub-folder: `josyn-backend/josyn-backend-global-config/`

**`JOSYN.Backend.GlobalConfig`** (NuGet library)
- `IGlobalConfig.cs` - contract; at minimum:
  - `string SessionStoreConnectionString`
  - `string JapServerExePath`
- `HardcodedGlobalConfig.cs` - PoC implementation with compile-time constants
  (same class of known limitation as the current fake session key in JAPServer)
- Dependencies: none (or `JOSYN.Foundation.ResultPattern` only)

This is a placeholder. The real implementation (file-based, registry, or company config system)
is a future concern. The interface stabilises the contract so all consumers are wired correctly
from day one.

Supporting: solution, `nuget.config`, `.local-build/`

Documentation:
- `ADR-003-global-config.md` in `josyn-platform/repos/josyn-backend/decisions/`
- Update `josyn-platform/repos/josyn-backend/overview.md`
- Add to `josyn-backend/.local-build/build.cmd` and `pack.cmd`

---

## Phase 3 - `JOSYN.Backend.SessionStarter` NuGet package

New Pattern-B sub-folder: `josyn-backend/josyn-backend-session-starter/`

**`JOSYN.Backend.SessionStarter`** (NuGet library)
- `ISessionStarter.cs` - contract: `Result<Guid> StartSession(string jobTypeName, string arguments)`
- `SessionStarter.cs` - allocates GUID, calls `ISessionStore.SaveNewSession`,
  reads `JapServerExePath` from `IGlobalConfig`, spawns `JAPServer.exe JOSYN-IPC <guid>`
- Dependencies: `JOSYN.Backend.SessionStore`, `JOSYN.Backend.GlobalConfig`, `JOSYN.Foundation.ResultPattern`

Note: NuGet producer only - no EXE here.
Future Category B executables (listener-service, ticker-service, cli-exe) are the consumers.

Supporting: solution, `nuget.config`, `.local-build/`

---

## Phase 4 - Demo round-trip (PoC)

New sub-folder: `josyn-backend/josyn-backend-demo/`

**`JOSYN.Backend.Demo.FakeSessionStarterConsumer`** (Console EXE)
- Wires up `HardcodedGlobalConfig` + real `SessionStore` (or in-memory stand-in) + `SessionStarter`
- Calls `StartSession("DemoJob", "<ini-arguments>")` with hardcoded demo INI args
- Spawns JAPServer, waits for exit, reads result back from SessionStore
- Dependencies: `JOSYN.Backend.SessionStarter`, `JOSYN.Backend.SessionStore`, `JOSYN.Backend.GlobalConfig`

**JAPServer wiring** in `josyn-backend/josyn-backend-jap-server/JOSYN.Jap.JAPServer/`:
- `Host.cs` - read `IGlobalConfig` for connection string; construct `SessionStore`; pass to `JAPServer`
- `JAPServer.cs` - replace fake methods:
  - `GetRawArguments()` -> `ISessionStore.GetSession(guid)` -> return `session.Arguments`
  - `PutRawResult(result)` -> `ISessionStore.UpdateSession(session with Result = result)`
- Add `JOSYN.Backend.SessionStore`, `JOSYN.Backend.GlobalConfig` as PackageReferences

### Demo round-trip proof
1. `FakeSessionStarterConsumer.exe` creates a session in SessionStore, spawns `JAPServer.exe JOSYN-IPC <guid>`
2. JAPServer reads `IGlobalConfig` for connection string, reads arguments from SessionStore
3. JAPServer writes result back to SessionStore
4. Consumer exe reads the completed session and prints the result

---

## Phase 5 - Architecture documentation

- `architecture/overview.md` - update component map and dependency chain
  (add SessionStore, GlobalConfig, SessionStarter; mark listener-service, ticker-service, cli-exe as placeholders)
- `repos/josyn-backend/overview.md` - reflect all new solutions

---

## Execution order

1. Phase 1 - SessionStore (standalone, no blockers)
2. Phase 2 - GlobalConfig (standalone, no blockers; can run in parallel with Phase 1)
3. Phase 3 - SessionStarter (requires Phase 1 + 2)
4. Phase 4 - Demo + JAPServer wiring (requires Phase 1 + 2 + 3)
5. Phase 5 - Docs (accompanies each phase or done at the end)
