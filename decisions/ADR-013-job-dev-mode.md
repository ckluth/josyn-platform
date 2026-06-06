# ADR-013 — Job Dev Mode: Running a Job Locally from the IDE

**Date:** 2026-06-06  
**Status:** Proposal — under construction; serves as basis for later refinement

---

## Context

A JOSYN job is a `.exe` spawned by `SessionStarter` (josyn-backend) via `JAPServer.exe`.
The invocation is:

```
job.exe JOSYN-IPC <sessionGUID>
```

`Core.Run(args)` in `josyn-job-host` expects this argument, parses the GUID, opens two named
pipes (`req-pipe-<guid>` / `res-pipe-<guid>`), and communicates with the running `JAPServer`
via `IJosynApplicationProtocol`. `JAPServer` in turn requires `SessionStore` (SQL Server)
and `GlobalConfig`.

**Problem:** When a developer presses F5 in the IDE:

1. the `JOSYN-IPC <guid>` argument is missing,
2. no `JAPServer` is running,
3. no database connection is established.

`Core.Run([])` fails at step 2: `JAPClient.CreateConnectedClient` cannot parse the GUID
→ `exit -1`. The developer cannot run or debug their job without the full infrastructure.

This is a fundamental DX problem: business logic inside `[JobEntryPoint]` is effectively
inaccessible unless the developer runs a complete local JOSYN instance.

---

## Friction Map

```
Developer presses F5
  → Core.Run([])
  → JAPClient.CreateConnectedClient([]) — GUID parse fails
  → LocalLog.Error(...)
  → exit -1
```

Three things are missing in the IDE context:
1. A running `JAPServer` (which requires SQL)
2. A session GUID that determines the pipe names
3. A source for the job's input arguments

---

## Options

### Option A — Dev-Mode Detection in `Core.Run`

**Idea:** `Core.Run` detects from the arguments that it is running in dev mode and switches
to a local execution path.

**Detection signal:** `args` is empty **or** `args[0] != "JOSYN-IPC"`.

```csharp
// Program.cs stays unchanged:
private static async Task<int> Main(string[] args) => await Core.Run(args);

// Core.Run — simplified concept:
public static async Task<int> Run(string[] args)
{
#if DEBUG
    if (args.Length == 0 || args[0] != "JOSYN-IPC")
        return await RunDev(args);
#endif
    // ... normal production path
}
```

**Dev path:**
- Reads job arguments from `dev-args.ini` in the current directory
- Executes the job (via the same `JobInvoker`)
- Writes the result to the console and to `dev-result.ini`
- No named pipe, no JAPServer, no SQL

**Developer workflow:**
1. Add `dev-args.ini` to the project (as `Copy if newer` to the output directory)
2. Press F5 → job runs, result on the console

**Pro:**
- Zero extra effort for the job developer
- Debugger works natively (no attach required)
- `dev-args.ini` is version-controllable alongside the job project
- `#if DEBUG` ensures no dev-path code in the release build

**Con:**
- `josyn-job-host` (a production library) explicitly knows about "dev mode" — questionable responsibility
- `#if DEBUG` ties dev mode to the build configuration rather than an explicit opt-in decision
- A job developer who accidentally omits `JOSYN-IPC` lands in the dev path instead of a clear error

---

### Option B — `JOSYN.JobHost.Dev` as a Separate NuGet Package

**Idea:** Dev-mode logic lives in its own package, referenced by the job project only in the
debug / development profile.

```xml
<!-- job project .csproj -->
<PackageReference Include="JOSYN.JobHost.Dev" Version="..." Condition="'$(Configuration)'=='Debug'" />
```

```csharp
// Program.cs (with conditional compilation):
#if DEBUG
private static async Task<int> Main(string[] args)
    => await DevCore.Run(args, fallbackArgsFile: "dev-args.ini");
#else
private static async Task<int> Main(string[] args) => await Core.Run(args);
#endif
```

`DevCore.Run`:
- If `args` = `JOSYN-IPC <guid>` → delegates to `Core.Run` (production path)
- Otherwise → local execution as described in Option A

**Pro:**
- `JOSYN.JobHost` (the core library) stays clean — no dev-mode paths
- Explicit opt-in: the job developer consciously chooses to use the dev library
- Clear package boundary: dev tooling has its own versioning and release lifecycle

**Con:**
- Job developer must consciously adjust `Program.cs` (slightly more friction than Option A)
- One more package in the platform portfolio — own repo candidate or sub-solution in `josyn-job-host`?
- Conditional `PackageReference` via `Condition` is unfamiliar and may cause confusion

---

### Option C — `DevJAPServer` as a Standalone Tool

**Idea:** A standalone executable (dotnet tool or local `.exe`) acts as a lightweight stub
server offering the same named-pipe interface as a real `JAPServer` — but without SQL,
without `SessionStore`, reading instead from a local configuration file.

**Flow:**
1. Developer starts `DevJAPServer.exe dev-args.ini`
2. Tool generates a fresh GUID and opens pipes under that GUID
3. Tool prints `JOSYN-IPC <guid>` to stdout
4. Developer configures IDE launch profile with that argument (or `.local-build/launch.debug.cmd` orchestrates both)
5. Job runs against the real named-pipe layer — no code changes in the job required
6. Tool receives `PutRawResult` / `PutError`, writes result to console + file, then exits

```
DevJAPServer.exe dev-args.ini
  → Pipes ready: JOSYN-IPC abc123...
  → [Developer starts job.exe JOSYN-IPC abc123...]
  → [GetRawArguments: returns contents of dev-args.ini]
  → [PutRawResult: writes dev-result.ini]
  → DevJAPServer exits
```

**Pro:**
- **Zero code changes** in `josyn-job-host` or the job project
- Genuinely realistic: the entire named-pipe layer is exercised
- Reusable across all job projects without job-specific configuration
- Testable: can serve as an integration-test tool for `JAPClient`

**Con:**
- Two processes must start in a coordinated manner — awkward for IDE F5
- IDE launch profiles with a dynamically generated GUID are difficult to configure
- Practical mainly for `launch.debug.cmd`, not for the IDE F5 workflow

---

### Option D — `.local-build/launch.debug.cmd` Without Code Changes

**Idea:** Rely exclusively on the existing `.local-build` convention system.
`launch.debug.cmd` in the job project:

1. Builds the job project in Debug
2. Starts `DevJAPServer.exe` (Option C), reads GUID from stdout
3. Starts `job.exe JOSYN-IPC <guid>` directly
4. Prints the result

**Pro:**
- Zero changes to libraries or package structure
- Consistent with the existing `.local-build` philosophy
- Hardcoded paths are explicitly permitted in `launch.*.cmd` (per `local-build.md`)

**Con:**
- No F5 debugging — the process is started as a standalone executable; the debugger must
  be attached manually
- Every job project needs its own `launch.debug.cmd`

---

## Comparison

| Criterion | Option A | Option B | Option C | Option D |
|-----------|----------|----------|----------|----------|
| F5 + debugger directly | ✅ | ✅ | ❌ | ❌ |
| No changes to `josyn-job-host` | ❌ | ✅ | ✅ | ✅ |
| No changes to job project | ✅ | ❌ | ✅ | ❌ |
| No infrastructure required | ✅ | ✅ | ✅ | ✅ |
| Real pipe layer exercised | ❌ | ❌ | ✅ | ✅ |
| Reusable without job-specific setup | ✅ | ❌ | ✅ | ❌ |
| Production binary stays clean | ⚠️ | ✅ | ✅ | ✅ |

**Initial leaning:** Two-track approach:
- **Short term → Option A** for immediate developer convenience: `#if DEBUG` guard in
  `Core.Run`, detection via empty / non-conforming args, `dev-args.ini` convention.
  Zero extra effort for the job developer; debugger works immediately.
- **Medium term → Option B** when the platform portfolio matures and a formal dev-tooling
  lifecycle becomes worthwhile. Option A can then be migrated to Option B internally.
- **Option C** is the right approach for integration-level testing (real pipe layer without
  SQL), but solves a different problem than IDE F5.
- **Option D** complements Option C for terminal-based workflows.

The decision has not been made yet — this ADR captures the problem space and the options.

---

## Open Questions (Refinement Basis)

1. **Where does dev-mode logic live?** Directly in `josyn-job-host` (Option A) or as a
   separate package (Option B)? This is the central design decision. Core question: is it
   acceptable for a production library to explicitly know about "dev mode"?

2. **Guard mechanism:** `#if DEBUG` (stripped at compile time, not runtime-configurable) vs.
   a runtime check (switchable, but always present in the binary) vs. explicit opt-in via
   `DevCore.Run` (Option B)? What is the right balance between safety and flexibility?

3. **Argument file format:** `dev-args.ini` is the natural choice because it mirrors the
   existing INI format of the PropertyBag serializer. Are there reasons to prefer JSON
   (better IDE support, nested structures)? Or does INI remain canonical?

4. **Result feedback:** Console output is often sufficient. Should `dev-result.ini` also be
   written so the result remains machine-readable without scrolling?

5. **`DevJAPServer` — own repo or sub-solution?** If Option C is implemented: does
   `DevJAPServer` belong in `josyn-backend` (as the dev counterpart to `JAPServer`), in
   `josyn-job-host` (as dev tooling for the job side), or in a new `josyn-dev-tools` repo?

6. **Relation to `josyn-sandbox`:** The sandbox repo is already the designated place for
   maintainer tooling (`deploy-maintainer.ps1`, demo jobs). Could `DevJAPServer` be hosted
   there without burdening the platform repos?

7. **Relation to test fakes:** `JOSYN.JobHost.Test` already contains stub implementations
   of `IJosynApplicationProtocol` (`JobInvokerTestSupport.cs`). Should those fakes be
   exposed as public test helpers (an Option B variant), rather than integrating dev mode
   into `Core.Run`?

8. **Multiple jobs, one `dev-args.ini`:** If a project has several `[JobEntryPoint]`
   candidates (e.g., during development), how is it controlled which one runs in dev mode?

---

## Relation to Other ADRs

- **ADR-012** (Maintainer Deployment): This ADR addresses the pre-deployment step —
  the developer in the loop before anything is deployed at all.
- **ADR-009** (Runtime Context Provider / Adapter): Dev mode must clarify whether adapter
  logic runs in the dev path or is bypassed.
- **ADR-004** (JAPServer Relocation): The separation of JAPServer protocol (josyn-jap) from
  JAPServer implementation (josyn-backend) is the reason why a dev stub is a fully separate
  piece of infrastructure rather than just a different configuration.
