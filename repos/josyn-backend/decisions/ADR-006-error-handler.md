# ADR-006 — ErrorHandler

**Date:** 2026-06-05
**Status:** Proposed

**Depends on:** ADR-008 (LocalLog Relocation to josyn-commons)

---

## Context

When the happy path is left, operators and maintainers need to know exactly what went wrong,
where, and why. The platform already has a stub: `JOSYN.Backend.ErrorHandler` with an
`IErrorHandler` contract and a `FileSystemErrorHandler` that writes to a temp file.
This stub must be replaced with a real, governed, durable error-handling component.

Several concerns must be resolved:

- Error information needs durable, queryable storage — not just a temp file
- Not all errors are equal: some are tied to a specific job execution; others are
  platform-level failures with no job context
- The error handler sits exclusively in the backend — `job.exe` and `josyn-job-host`
  cannot reference it; a bridging mechanism already exists (`PutError` over IPC)
- When the error handler itself fails, error information must not be silently lost

---

## Decision

### 1. Error records are a first-class Storage Realm domain

Error records are durable platform state. They belong in the JOSYN Storage Realm
(SQL Server, `josyn` schema) — the same database and schema already used by
`SessionStore` and `JobRegistry`.

The new storage domain:

| Domain | Package | Table | Status |
|--------|---------|-------|--------|
| Error records | `JOSYN.Backend.ErrorHandler` | `josyn.ErrorStore` | This ADR |

This updates the domain table in ADR-007 and `architecture/storage.md`.

### 2. Two error kinds — distinguished by nullable context fields

Errors are classified by the context they carry:

| Kind | `JobName` | `SessionGuid` | Example |
|------|-----------|---------------|---------|
| System error | `null` | `null` | Ticker fails to read job registry |
| Job error, no session | set | `null` | SessionStarter fails before session row is written |
| Job error, with session | set | set | Job execution fails; JAPServer receives PutError |

There is no explicit enum column. The nullable pattern is sufficient — the kind is
inferrable directly from the data.

### 3. Error record schema — `IErrorRecord`

```csharp
public interface IErrorRecord
{
    Guid            Id                 { get; init; }  // generated on storage
    DateTimeOffset  OccurredAt         { get; init; }
    string          Causer             { get; init; }  // process or component name
    string          Message            { get; init; }
    string?         CallStack          { get; init; }  // Result chain or stack trace
    string?         ExceptionDetails   { get; init; }  // serialized exception, if any
    string?         JobName            { get; init; }  // null for system errors
    Guid?           SessionGuid        { get; init; }  // null if no session established
}
```

Concrete type follows the platform `IxxxRecord` convention (ADR-007):

```csharp
public sealed record ErrorRecord(...) : IErrorRecord;
```

### 4. `IErrorHandler` contract

`IErrorHandler` is a fire-and-forget contract. It never throws. Callers do not inspect
the outcome — the handler is responsible for its own fallback.

```csharp
public interface IErrorHandler
{
    void Handle(
        string          message,
        string?         callStack        = null,
        string?         exceptionDetails = null,
        string?         jobName          = null,
        Guid?           sessionGuid      = null,
        [CallerMemberName] string caller = "");
}
```

`[CallerMemberName]` is captured automatically at the call site — callers do not provide
it explicitly. It identifies the method that observed the terminal failure and invoked the
handler. `[CallerFilePath]` is deliberately excluded: it would store absolute machine-specific
source paths in durable SQL records, producing noisy, environment-dependent data with no
diagnostic value beyond what `caller` already provides.

The existing `FileSystemErrorHandler` is replaced by `SqlErrorHandler`, which:
1. Constructs an `ErrorRecord` from the call site and supplied parameters
2. Persists it to `josyn.ErrorStore` via EF Core
3. On any storage failure, falls back to `LocalLog` (`JOSYN.Commons.Log`) silently

```csharp
public sealed class SqlErrorHandler(string connectionString) : IErrorHandler
```

### 5. Where the error handler is called — placement rule

`JOSYN.Backend.ErrorHandler` is a **backend-only component**. It cannot be referenced
by `josyn-jap`, `josyn-job-host`, or `job.exe` — the dependency direction forbids it.

The placement rule:

> The error handler is called at every point in backend code where a `Result` chain
> terminates without a further caller — and when a `PutError` message arrives from
> a job via IPC.

Concretely:

| Call site | Error kind | Notes |
|-----------|------------|-------|
| `JAPServer.PutError()` receives a report | Job error, with session | Bridge for all job-side failures |
| `JAPServer` observes unexpected job process exit | Job error, with session | Job died before it could call PutError |
| `josyn-backend` orchestration entry points (ticker, service, session-starter) | Job or system | Depends on whether job context is known |

Within a process, the `Result` pattern propagates failures. The error handler fires only
at the terminal boundary — where no further caller exists to receive the `Result`.

`job.exe` is explicitly outside the error handler's reach. Its contract is:
`LocalLog` for local visibility; `PutError` over IPC to hand off to the backend.
`PutError` is the bridge — it is the point where a job-side failure enters the backend
and reaches `IErrorHandler`.

### 5a. JobName and SessionGuid enrichment at the PutError call site

When JAPServer receives `PutError`, the `IErrorReport` payload contains no `JobName` or
`SessionGuid` — the job does not know its own registered name, and the IPC wire type
(`JOSYN.Jap.Shared.Contract`) carries no backend context.

Enrichment rule:

- `SessionGuid` — known to JAPServer at construction (passed by `SessionStarter` at spawn time)
- `JobName` — passed to JAPServer as a constructor argument by `SessionStarter` at spawn time,
  alongside `SessionGuid`

`SessionStarter` holds both values when it spawns `JAPServer.exe`. Passing them at
construction is simpler and more resilient than looking up the session record at error time —
a DB lookup on a failure path adds latency and a new failure mode when the DB is the cause
of the error.

### 6. Fallback strategy — two-tier persistence

| Tier | Mechanism | When used |
|------|-----------|-----------|
| Primary | `josyn.ErrorStore` (SQL) | Normal operation |
| Fallback | `LocalLog` (`JOSYN.Commons.Log`) | SQL unavailable or storage call fails |

The handler swallows its own failures — it never propagates them to the caller.
This matches the `LocalLog` silent-failure policy and prevents error-handling from
becoming an error source.

Note: the fallback depends on `JOSYN.Commons.Log` being available (ADR-008 prerequisite).

### 7. `FileSystemErrorHandler` is removed

The stub `FileSystemErrorHandler` is deleted. `SqlErrorHandler` is the sole production
implementation. No file-based permanent implementation is retained — `LocalLog` fallback
serves as the safety net.

### 8. Deferred — propagation and access (follow-on ADR)

The following concerns are explicitly out of scope for this ADR:

- **Active propagation:** notifying operators via email, messenger, or other channels
  when errors are recorded
- **Access and search:** viewing, filtering, printing, and exporting error records via
  an operator shell or similar tool

A follow-on ADR for these concerns depends on this ADR being accepted first. The
`josyn.ErrorStore` table and `IErrorRecord` schema defined here are the foundation
for that work.

---

## Decision challenge — objections and rebuttals

**Attacker:** Error records belong in a log file, not a SQL table. The DB is often the cause
of backend failures — storing error records in the same database risks losing exactly the
errors you most need to see.

**Defender:** The two-tier fallback directly addresses this. SQL is the primary store for
normal operation: it makes error records queryable by job, session, time range, and causer —
something a flat file can never offer. `LocalLog` is the fallback precisely for the case where
SQL is unavailable. The combination means you never depend on SQL alone, but you gain all its
advantages when it is healthy.

---

**Attacker:** An explicit `ErrorKind` enum column would make error classification
self-documenting in the database. Querying "all system errors" requires no knowledge of
the nullable convention — just `WHERE ErrorKind = 'System'`.

**Defender:** The nullable fields `JobName` and `SessionGuid` *are* the classification — they
carry the actual context, not just a label. An enum column would duplicate information already
present, creating a surface for inconsistency: what does `ErrorKind = 'Job'` with
`JobName = null` mean? The nullable pattern is stricter — the data itself encodes the kind
with no redundancy to maintain.

---

**Attacker:** `IErrorHandler` with `void Handle()` is fire-and-forget, but SQL persistence
is async. A `void` contract forces either synchronous blocking I/O or hidden background tasks
— both are problematic in an async-first platform.

**Defender:** This is a valid tension that the ADR acknowledges. The resolution is an
implementation concern, not a contract concern: `SqlErrorHandler` may perform a synchronous
blocking write (acceptable for a rarely-called error path) or marshal to a background task
with appropriate shutdown coordination. The contract stays `void` because the caller's
contract with the error handler is genuinely fire-and-forget — callers should never be
blocked by or made aware of persistence mechanics. The implementation shape is decided when
the handler is built, informed by the async characteristics of EF Core in this context.

---

**Attacker:** The error handler is backend-only, so job-side errors are only persisted when
`PutError` arrives over IPC. If IPC fails — the pipe is broken, the job crashes before
calling `PutError` — the error is only in `LocalLog` on the job machine. That's a significant
gap in coverage.

**Defender:** This is the correct and honest boundary. When IPC fails, the backend has no
channel to the job — it cannot reach in and extract error information. `LocalLog` on the job
side is the last-resort record for those cases. The backend *can* observe the symptom (job
process exits with non-zero code, no `PutRawResult` recorded in the session store) and record
a system-level error for that session — but the job's internal failure detail is only in
`LocalLog`. This is acknowledged in the placement rule: JAPServer observing an unexpected job
exit is itself a call site for `Handle()`.

---

**Attacker:** The `Result` pattern already propagates call-stack information through the chain.
Why capture `[CallerMemberName]` / `[CallerFilePath]` at the `Handle()` call site too?

**Defender:** By the time `Handle()` is called, the `Result` chain has terminated — its
`CallStack` is passed in as the `callStack` parameter. The `[CallerMemberName]` capture is
different: it identifies the backend component that made the decision to call `Handle()`.
That is the `Causer` field — not the internal failure origin, but the architectural boundary
where the error was observed and handed to the handler. These are complementary, not redundant.

---

## Consequences

- Error information survives process restarts and is queryable by job, session,
  time range, and causer
- The two-tier fallback guarantees that an error is never silently lost, even when
  the database is unavailable
- `IErrorHandler` is fire-and-forget: callers are never burdened with error-handler
  failures
- `[CallerMemberName]` / `[CallerFilePath]` give stored records precise provenance
  without requiring call sites to self-identify
- The placement rule makes it unambiguous where `Handle()` must be called in backend code
- `PutError` as the IPC bridge keeps the dependency boundary clean: job-side code never
  references backend packages
- `FileSystemErrorHandler` is replaced; the temp-file stub is gone
- The storage domain table in ADR-007 and `architecture/storage.md` is updated to
  reflect `josyn.ErrorStore` as an active domain
