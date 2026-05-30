# josyn-jap

**Role:** Per-session JAP protocol server — handles a single job session from argument delivery
to result/error collection. Contains the shared protocol contract, a shared logger, and the
backend server executable.

**Location:** `C:\Users\chris\OneDrive\DevGit\josyn-jap`
**Version:** `1.0.0-preview01`

---

## Structure

```
josyn-jap/
├── josyn-jap-shared/             ← NuGet libraries (two packages)
│   ├── JOSYN.Jap.Shared.Contract/
│   ├── JOSYN.Jap.Shared.Log/
│   └── JOSYN.Jap.Shared.Log.Test/
└── josyn-jap-japserver/          ← Backend EXE
    └── JOSYN.Jap.JAPServer/
```

---

## JOSYN.Jap.Shared.Contract

**Purpose:** Transport-agnostic definition of the JOSYN Application Protocol (JAP).
Shared between JAPServer (implements it) and JobHost (calls it via JAPClient).

**Dependencies:** `JOSYN.Foundation.ResultPattern`

### Key types

```csharp
// The protocol — three async operations
public interface IJosynApplicationProtocol
{
    Task<Result<string>> GetRawArguments();
    Task<Result> PutRawResult(string result);
    Task<Result> PutError(string serializedError);
}

// Error DTO — immutable, serializable via PropertyBag
public sealed record ErrorReport(
    string Causer,
    string Message,
    string? CallStack,
    string? ExceptionDetails,
    DateTimeOffset OccurredAt)
    : IErrorReport;

public interface IErrorReport
{
    string Causer { get; init; }
    string Message { get; init; }
    string? CallStack { get; init; }
    string? ExceptionDetails { get; init; }
    DateTimeOffset OccurredAt { get; init; }
}
```

### Protocol semantics

| Operation | Direction | Purpose |
|-----------|-----------|---------|
| `GetRawArguments()` | Server → Job | Returns serialized job arguments (INI or JSON string) |
| `PutRawResult(string)` | Job → Server | Delivers serialized job result |
| `PutError(string)` | Job → Server | Delivers serialized `ErrorReport` on failure |

All three return `Result` / `Result<T>` — no exceptions cross the protocol boundary.

---

## JOSYN.Jap.Shared.Log

**Purpose:** Process-local file logger. Used by both JAPServer and job executables (via JobHost).

**Dependencies:** `JOSYN.Foundation.ResultPattern`

### Key API

```csharp
// Static abstract interface (C# 11)
public interface ILocalLog
{
    static abstract string LogDirectory { get; set; }
    static abstract bool EnableConsoleOutput { get; set; }
    static abstract void WriteError(string message, ...);
    static abstract void WriteInfo(string message, ...);
}

// Implementation
public static class LocalLog : ILocalLog
{
    public static string LogDirectory { get; set; }           // set at startup
    public static bool EnableConsoleOutput { get; set; }      // true in DEBUG builds

    public static void WriteError(string message, ...);
    public static void WriteError(Result result);                   // overload for Result
    public static void WriteError(string causer, string message);  // writes to subdirectory

    public static void WriteInfo(string message);
    public static void WriteInfo(string causer, string message);
}
```

### Log file layout

```
<LogDirectory>/
├── <causer>/
│   └── <yyyy-MM-dd>.log     ← causer-specific log (from WriteError(causer, message))
└── <yyyy-MM-dd>.log         ← process-level log (from WriteError(message))
```

Log entries include: timestamp, header, message, optional CallStack section, optional Exception section.

**Silent failure policy:** I/O errors in the logger are swallowed — it never crashes the host.

---

## JOSYN.Jap.JAPServer

**Purpose:** Backend server executable. Listens on named pipes (JIP), receives JAP requests from job executables, dispatches to the `IJosynApplicationProtocol` implementation.

**Dependencies:** `JOSYN.Foundation.JIP`, `JOSYN.Foundation.PropertyBag`, `JOSYN.Foundation.ResultPattern`, `JOSYN.Jap.Shared.Contract`, `JOSYN.Jap.Shared.Log`

**Type:** `net10.0` Console EXE

### CLI contract

```
JOSYN.Jap.JAPServer.exe JOSYN-IPC <sessionGUID>
```

Exit codes:
- `0` — success
- `1` — fatal error (missing key, IPC failure, unhandled exception)

### Structure

```csharp
// Program.cs — thin entry point
static async Task<int> Main(string[] args) => await Host.Run(args);

// Host.cs — lifecycle and dispatch
static class Host
{
    static Task<int> Run(string[] args)
    {
        // 1. Parse session key from args
        // 2. Create JAPServer instance
        // 3. Register all IJosynApplicationProtocol methods with JipDispatcher
        // 4. Start PipesServer — blocking until ESC or fatal error
    }
}

// JAPServer.cs — IJosynApplicationProtocol implementation
sealed class JAPServer : IJosynApplicationProtocol
{
    Task<Result<string>> GetRawArguments();        // PoC: returns fake INI data
    Task<Result> PutRawResult(string result);      // PoC: prints to console
    Task<Result> PutError(string serializedError); // deserializes ErrorReport → LocalLog
}
```

### JIP dispatcher wiring

```csharp
_jipDispatcher.RegisterAll<IJosynApplicationProtocol>(jAPServer);
```

`RegisterAll` auto-discovers all methods on `IJosynApplicationProtocol` via reflection and maps them to `what` strings matching method names.

### PoC limitations

- `GetRawArguments` returns hardcoded fake INI data (`FakeReadArgumentsFromFile`)
- Demo session key: `dea5611d-d740-437f-ad93-7a5dc5ae4299` in `launchSettings.json`
- Machine-specific build output path in `Directory.Build.props`

---

## Build & Package

```
josyn-jap-shared/.local-build/pack.cmd      ← packs Contract + Log to ../../local-packages/
josyn-jap-japserver/.local-build/build.cmd
```

License: MIT | Company: HAEVG AG | Target: net10.0

---

## Sanity Notes

### Structural specifics
- **Multi-subdirectory repo**: `josyn-jap-shared/` (two NuGet packages) and `josyn-jap-japserver/` (EXE). Each subdirectory has its own `.local-build/` scripts and is evaluated independently.
- `JOSYN.Jap.Shared.Contract` and `JOSYN.Jap.Shared.Log` are NuGet libraries — must have `GenerateDocumentationFile`, NuGet metadata, `icon.png`.
- `JOSYN.Jap.JAPServer` is a Console EXE — must **not** have `GenerateDocumentationFile` or NuGet metadata.

### Dependency constraints
- Shared packages (`Contract`, `Log`) may only reference `JOSYN.Foundation.ResultPattern`. Any reference to `PropertyBag`, `JIP`, or any `josyn-jap` package is a violation.
- JAPServer may reference: `JOSYN.Foundation.JIP`, `JOSYN.Foundation.PropertyBag`, `JOSYN.Foundation.ResultPattern`, `JOSYN.Jap.Shared.Contract`, `JOSYN.Jap.Shared.Log`.

### Known exceptions (not violations)
- `JAPServer.cs` uses `sealed class` (not `static`) — correct; it has instance state implementing `IJosynApplicationProtocol`.
- `GetRawArguments()` returns hardcoded fake INI data — known PoC limitation, not a violation in the current phase.
- Demo session key in `launchSettings.json` — PoC convenience, not a violation.

