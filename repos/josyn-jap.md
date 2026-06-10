# josyn-jap

**Role:** JAP protocol contracts — the single source of truth for the contract shared between
`JOSYN.Jap.JAPServer` (server, in `josyn-backend`) and `JOSYN.JobHost` (client). Contains the
shared protocol contract (`JOSYN.Jap.Contract`). The process-local logger has been
relocated to `JOSYN.Commons.Log` per ADR-008.

**Location:** `C:\DevGit\josyn-jap`
**Version:** `1.0.0-preview01`

---

## Structure

```
josyn-jap/
└── josyn-jap-shared/             ← NuGet library (one package)
    └── JOSYN.Jap.Contract/
```

> `JOSYN.Jap.JAPServer` (the EXE) was relocated to `josyn-backend` per ADR-004.
> `JOSYN.Jap.Shared.Log` was relocated to `josyn-commons` as `JOSYN.Commons.Log` per ADR-008.

---

## JOSYN.Jap.Contract

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

## Build & Package

```
josyn-jap-shared/.local-build/pack.cmd      ← packs Contract to ../../local-packages/
```

License: MIT | Company: HAEVG AG | Target: net10.0

---

## Sanity Notes

### Structural specifics
- **Single-subdirectory repo**: `josyn-jap-shared/` contains one NuGet package. Evaluated as one unit.
- `JOSYN.Jap.Contract` is a NuGet library — must have `GenerateDocumentationFile`, NuGet metadata, `icon.png`.

### Dependency constraints
- `JOSYN.Jap.Contract` may only reference `JOSYN.Foundation.ResultPattern`. Any reference to `PropertyBag`, `JIP`, or any other package is a violation.

### Known exceptions (not violations)
- No EXE project in this repo — `JOSYN.Jap.JAPServer` was relocated to `josyn-backend` per ADR-004.
- `JOSYN.Jap.Shared.Log` was retired and relocated to `JOSYN.Commons.Log` per ADR-008.

