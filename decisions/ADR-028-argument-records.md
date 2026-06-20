# ADR-028 — ArgumentRecord: Named Argument Payloads in the Job Registry

**Date:** 2026-06-19
**Status:** Draft
**Extends:** ADR-007B-02 (JobRegistry)

---

## Context

The job registry (ADR-007B-02) records, per job, the two facts needed to launch a session:
the job's platform-wide unique name and the OS-level technical user identity under which
its process runs. It does not record *what arguments* to pass when launching a session.

Currently, argument payloads live only as INI files on the maintainer's filesystem, under
a convention-based path:

```
BackendRoot\JobRepository\<JobName>\local-arguments\<name>.ini
```

The CLI orchestrator knows this convention and can resolve arguments at launch time. No
other orchestrator can. In particular, `TimeScheduler.exe` — which fires sessions without
any human present — has no governed path to an argument payload. The current stub hardcodes
both the job name and a fixed file path.

ADR-027 (JobSchedule Store) formally deferred the argument record definition as a
prerequisite and introduced `ArgumentRecordName` as an opaque string reference. This ADR
closes that gap.

---

## Decision

### 1. `ArgumentRecord` — the concept

An `ArgumentRecord` is a named INI-serialized argument payload owned by a registered job.
It is the governed, database-stored replacement for the filesystem `*.ini` argument file.

Each `ArgumentRecord` is identified by its **name** within the scope of its owning job.
The name is operator-chosen, free-form, and has no platform-enforced semantics.

**Convention:** the name `"default"` is used for the standard single-payload case. This
mirrors the existing filesystem convention (`arguments-default.ini`). The platform does not
enforce this name or assign it special behaviour — it is simply the obvious choice when a
job has exactly one argument record.

### 2. Schema extension — `josyn.ArgumentRecords`

A new table `josyn.ArgumentRecords` is added to the JOSYN storage realm, owned by
`JOSYN.Backend.JobRegistry`.

| Column | Type | Constraints |
|--------|------|-------------|
| `JobName` | `nvarchar(256)` | PK (part 1); FK → `josyn.JobRegistrations.Name` |
| `Name` | `nvarchar(256)` | PK (part 2) |
| `Content` | `nvarchar(max)` | NOT NULL; INI-serialized argument payload, stored verbatim |

The composite PK `(JobName, Name)` enforces uniqueness of names within a job and makes
individual records directly addressable without a surrogate key.

`Content` is stored verbatim — the registry is not aware of the INI format. Parsing is the
caller's responsibility, consistent with how `josyn.JobScheduleEntries` stores JSONC
verbatim (ADR-027).

### 3. Package extension — `JOSYN.Backend.JobRegistry`

The existing `JOSYN.Backend.JobRegistry` package is extended. No new package is introduced.
Argument records are part of the job's registration data — the registry grows from "job
identity" to "job identity and invocation payloads."

#### New public contract — `IArgumentRecord`

```csharp
public interface IArgumentRecord
{
    string JobName  { get; init; }
    string Name     { get; init; }
    string Content  { get; init; }
}
```

#### `IJobRegistrationRecord` — extended

`ArgumentRecords` is added to the existing registration record:

```csharp
public interface IJobRegistrationRecord
{
    string                              Name             { get; init; }
    string                              TechnicalUserName { get; init; }
    IReadOnlyList<IArgumentRecord>      ArgumentRecords  { get; init; }
}
```

Argument records are always loaded together with the registration. The list is empty (not
null) for a job that has no stored argument records. Callers that only need identity fields
pay the cost of an extra left join — acceptable given the expected row counts.

#### `IJobRegistry` — new lookup method

A targeted lookup is added alongside the existing `GetByName` and `GetAll`:

```csharp
public interface IJobRegistry
{
    Result<IJobRegistrationRecord>               GetByName(string name);
    Result<IReadOnlyList<IJobRegistrationRecord>> GetAll();
    Result<IArgumentRecord>                      GetArgument(string jobName, string argumentName);
}
```

`GetArgument` is the hot path for `TimeScheduler`: given a `JobName` + `ArgumentRecordName`
from a `JobScheduleEntry` (ADR-027), return the payload without loading the full registration.
Returns `Result.Error` if the job or the named record does not exist.

### 4. Filesystem path — not removed, scope narrowed

The filesystem convention (`local-arguments\*.ini`) is **not removed**. It remains the
authoring and development path: a job developer creates and edits argument files locally,
and the CLI can still load them by path.

The database is the **runtime path**: `TimeScheduler` and any other orchestrator that cannot
rely on filesystem conventions use `IJobRegistry.GetArgument()`. This creates a dual path
during the current phase.

How filesystem argument files are imported into the database (seeding, tooling, or a
migration helper) is **not decided here**. It is a tooling concern deferred to
`josyn-toolbox`.

---

## Alternatives considered

### A separate `JOSYN.Backend.ArgumentStore` package

Argument records could live in a dedicated package, keeping `JobRegistry` focused on
identity only.

Rejected: argument records are not independently meaningful — they exist only in the context
of a job registration. The registry already owns the job's name, technical user, and (after
ADR-027) is the anchor for schedule entries. Splitting argument payloads into a separate
package creates an artificial boundary that consumers would always cross together.

### Nullable name — unnamed default record

Allow one record per job to carry a null name (the implicit default), with additional
records carrying explicit names.

Rejected: a nullable primary key component is awkward in SQL (`NULL ≠ NULL` in composite
keys) and in C#. Option A (always named, `"default"` by convention) is uniform and equally
clear without the nullable-key complexity.

### Store parsed INI as structured columns

Normalise the INI content into a relational form (key-value rows) rather than storing it
verbatim.

Rejected: it would require the registry to understand the INI format, coupling it to a
serialization concern it has no business owning. Storing verbatim and delegating parsing to
callers is consistent with how `ScheduleDefinition` JSONC is stored in ADR-027.

---

## Open questions

1. **Import tooling** — how are existing filesystem `*.ini` argument files imported into
   `josyn.ArgumentRecords`? A `josyn-toolbox` command (`import-arguments`?) is anticipated
   but not designed here.

2. **Write operations** — adding, updating, and removing argument records via a governed
   interface (CLI or admin UI) is deferred. For now, records are inserted directly via SQL
   or the bootstrap script.

3. **Content format validation** — `Content` is stored and returned verbatim. Should the
   registry validate that `Content` is well-formed INI before persisting? Currently: no.
   The registry is format-agnostic; validation is the caller's responsibility.

---

## Consequences

- `JOSYN.Backend.JobRegistry` gains a new table `josyn.ArgumentRecords`, a new
  `IArgumentRecord` contract, an extended `IJobRegistrationRecord`, and a new
  `GetArgument()` method on `IJobRegistry`.
- `josyn-backend/db/migrations/` gains a new migration script for `josyn.ArgumentRecords`
  (next available version after ADR-027's migration).
- `architecture/storage.md` — the Domains table entry for `Job registry` expands its
  Table column to `josyn.JobRegistrations + josyn.ArgumentRecords`.
- ADR-027 open question #2 (argument record definition) is closed by this ADR.
- `TimeScheduler.exe` can now resolve `ArgumentRecordName` → `Content` via
  `IJobRegistry.GetArgument()` without any filesystem access.

---

## Relation to other ADRs

- **ADR-007B-02** (JobRegistry): this ADR extends the registry's schema and package surface.
  All existing conventions (EF Core `internal sealed`, `IxxxRecord` naming, connection
  string injection) are preserved.
- **ADR-027** (JobSchedule Store): `JobScheduleEntry.ArgumentRecordName` is resolved via
  `IJobRegistry.GetArgument()` defined here. ADR-027 open question #2 is closed.
- **ADR-026** (Schedule Definition Language): no direct relation; noted for consistency
  — both ADR-026 and this ADR store domain-specific text verbatim in `nvarchar(max)` columns
  and delegate parsing to the appropriate library.
