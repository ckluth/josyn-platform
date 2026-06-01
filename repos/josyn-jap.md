# josyn-jap

**Role:** JAP protocol contracts — the single source of truth for the contract shared between
`JOSYN.Jap.JAPServer` (server, in `josyn-backend`) and `JOSYN.JobHost` (client). Contains the
shared protocol contract and the shared process-local logger.

**Location:** `C:\Users\chris\OneDrive\DevGit\josyn-jap`
**Version:** `1.0.0-preview01`

---

## Structure

```
josyn-jap/
└── josyn-jap-shared/             ← NuGet libraries (two packages)
    ├── JOSYN.Jap.Shared.Contract/
    ├── JOSYN.Jap.Shared.Log/
    └── JOSYN.Jap.Shared.Log.Test/
```

> `JOSYN.Jap.JAPServer` (the EXE) was relocated to `josyn-backend` per ADR-004.
> See `decisions/ADR-004-japserver-relocation.md`.

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

## Build & Package

```
josyn-jap-shared/.local-build/pack.cmd      ← packs Contract + Log to ../../local-packages/
```

License: MIT | Company: HAEVG AG | Target: net10.0

---

## Sanity Notes

### Structural specifics
- **Single-subdirectory repo**: `josyn-jap-shared/` contains the two NuGet packages. Evaluated as one unit.
- `JOSYN.Jap.Shared.Contract` and `JOSYN.Jap.Shared.Log` are NuGet libraries — must have `GenerateDocumentationFile`, NuGet metadata, `icon.png`.

### Dependency constraints
- Shared packages (`Contract`, `Log`) may only reference `JOSYN.Foundation.ResultPattern`. Any reference to `PropertyBag`, `JIP`, or any other package is a violation.

### Known exceptions (not violations)
- No EXE project in this repo — `JOSYN.Jap.JAPServer` was relocated to `josyn-backend` per ADR-004.

