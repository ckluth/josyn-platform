# ADR-027 — JobSchedule Store: Scheduling Configuration for Registered Jobs

**Date:** 2026-06-19
**Status:** Draft

---

## Context

ADR-024 established `TimeScheduler.exe` as the orchestrator responsible for time-based job
launches. ADR-026 defined the schedule definition language — the JSONC rule format that
describes *when* a job should run.

Neither ADR decided where scheduling configuration is stored for a concrete job, or how
`TimeScheduler` maps a job to its schedule and to the arguments it should receive.

The current `TimeScheduler` stub hardcodes both: it always fires `"Contoso.DemoProduct.DemoJob"`
with the contents of a fixed `arguments-default.ini` file. That must be replaced by a real,
queryable data structure.

### What is missing

Three things are absent from the platform:

1. **Named argument records in the job registry.** Argument records — the INI-serialized
   argument payloads passed to a job session — currently exist only as files on the
   maintainer's filesystem (`JobRepository\<JobName>\local-arguments\arguments-default.ini`).
   The CLI orchestrator can find them by convention; a scheduled execution cannot. The job
   registry must be extended to carry 0 to n named argument records per job, stored in the
   database, so that any orchestrator — not only the CLI — can resolve an argument record
   by name at runtime. This is a prerequisite for scheduled execution and is the subject of
   a separate ADR. This ADR treats `ArgumentRecordName` as an opaque reference string that
   will be resolved once that ADR is in place.

2. **A storage structure** that records, for each registered job, the set of schedules it
   should be evaluated against — including which argument record to pass when the schedule
   fires, and whether the job is currently suspended.

3. **An evaluation algorithm** in `TimeScheduler` that queries that structure, compares each
   active schedule against the current machine datetime, and launches a session for every
   entry that is due.

This ADR defines item 2 and the contract surface that item 3 will consume. The evaluation
algorithm itself is an implementation concern of `TimeScheduler` and is not specified here
beyond what the contract implies. Item 1 is acknowledged as a prerequisite and is deferred
to its own ADR.

---

## Decision

### 1. The `JobSchedule` concept

A `JobSchedule` is the per-job scheduling record: the authoritative source of truth for
*when* a job runs and *how* it is invoked from a time-based trigger.

Every registered job (`IJobRegistrationRecord` from ADR-007B-02) may have at most one
`JobSchedule`. A job with no `JobSchedule` is never launched by `TimeScheduler`.

A `JobSchedule` consists of:

- A **job name** — the foreign key into `JobRegistry` (`IJobRegistrationRecord.Name`).
- One or more **entries** — each entry defines one independently scheduled invocation.
- A **suspension state** — an operator-level switch that can suppress all entries for
  this job without modifying the entries themselves (see §3).

### 2. `JobScheduleEntry` — one independently scheduled invocation

Each entry within a `JobSchedule` carries:

| Field | Type | Description |
|-------|------|-------------|
| `ArgumentRecordName` | string | The name of the argument record to pass to `SessionLauncher` when this entry fires. This name also serves as the identity of the entry within the `JobSchedule`. Two entries for the same job must not share an `ArgumentRecordName`. |
| `ScheduleDefinition` | string (JSONC) | The inline ADR-026 schedule definition text. Parsed by `JOSYN.Commons.Schedule` at evaluation time. |

`ArgumentRecordName` is both the payload key (which argument set to use) and the natural
identity of the entry. No separate display name is needed — entries are distinguished by
what they invoke.

> **Argument records** — the definition of what an argument record is, where it is stored,
> and how it is resolved by name — is not decided here. That is a separate ADR. For this ADR,
> `ArgumentRecordName` is an opaque string that `TimeScheduler` passes through to the session
> launch request.

### 3. Suspension state — per job, two fields

| Field | Type | Description |
|-------|------|-------------|
| `Suspended` | bool | When `true`, no entry in this `JobSchedule` is evaluated. `TimeScheduler` skips the job entirely for this tick. |
| `SuspendedUntil` | `DateOnly?` | Optional. When `Suspended` is `true` and `SuspendedUntil` is set, `TimeScheduler` treats the job as not-suspended once the server's local date passes this value. The auto-resume is evaluated by comparing `DateOnly.FromDateTime(DateTime.Now)` against `SuspendedUntil`. When the suspension lifts automatically, `Suspended` is **not** written back to the database — the evaluator treats a past `SuspendedUntil` as inactive. |

`SuspendedUntil` without `Suspended = true` is a validation error. The flag is the gate;
the date is the optional auto-lift.

Suspension is an operator-level override. It does not modify or invalidate the schedule
entries — operators can suspend and resume a job without touching the `ScheduleDefinition`
content.

Per-entry suspension is **not** supported in this version. If the need arises, it is an
additive change (new nullable columns on the entry row) and does not require a redesign of
this structure.

### 4. Storage — `josyn.JobSchedules` and `josyn.JobScheduleEntries`

`JobSchedule` data is stored in the JOSYN storage realm (ADR-007) as two new tables in the
`josyn` schema, following the existing naming and migration conventions.

#### `josyn.JobSchedules`

| Column | Type | Constraints |
|--------|------|-------------|
| `JobName` | `nvarchar(256)` | PK; FK → `josyn.JobRegistry.Name` |
| `Suspended` | `bit` | NOT NULL; default `0` |
| `SuspendedUntil` | `date` | NULL |

#### `josyn.JobScheduleEntries`

| Column | Type | Constraints |
|--------|------|-------------|
| `JobName` | `nvarchar(256)` | PK (part 1); FK → `josyn.JobSchedules.JobName` |
| `ArgumentRecordName` | `nvarchar(256)` | PK (part 2) |
| `ScheduleDefinition` | `nvarchar(max)` | NOT NULL; stores the JSONC text verbatim |

The composite PK `(JobName, ArgumentRecordName)` enforces the uniqueness rule from §2.

#### NuGet package — `JOSYN.Backend.JobScheduleStore`

A new Category A NuGet package `JOSYN.Backend.JobScheduleStore` is introduced in
`josyn-backend`, following the building block model (ADR-005B-01). It owns:

- `IJobScheduleEntryRecord` — the public data contract for one entry
- `IJobScheduleRecord` — the public data contract for one `JobSchedule` (header + entries)
- `IJobScheduleStore` — the query interface consumed by `TimeScheduler`
- `SqlJobScheduleStore` — the EF Core production implementation

Public interfaces:

```csharp
public interface IJobScheduleEntryRecord
{
    string ArgumentRecordName  { get; init; }
    string ScheduleDefinition  { get; init; }
}

public interface IJobScheduleRecord
{
    string                                    JobName        { get; init; }
    bool                                      Suspended      { get; init; }
    DateOnly?                                 SuspendedUntil { get; init; }
    IReadOnlyList<IJobScheduleEntryRecord>    Entries        { get; init; }
}

public interface IJobScheduleStore
{
    Result<IReadOnlyList<IJobScheduleRecord>> GetAll();
}
```

`GetAll()` returns every `JobSchedule` row together with its entries. `TimeScheduler` loads
the full set once per invocation and evaluates locally — no per-job query round-trip.

EF Core internals are `internal sealed` per platform convention. Constructor injection of
the connection string follows the `SqlJobRegistry` precedent (ADR-007B-02, §4).

### 5. `TimeScheduler` evaluation sketch

The evaluation algorithm is owned by `TimeScheduler` and not specified in detail here. The
contract implies the following sequence:

1. Call `IJobScheduleStore.GetAll()`.
2. For each `IJobScheduleRecord`, evaluate suspension state:
   - If `Suspended` is `true` and `SuspendedUntil` is null, or `SuspendedUntil` is in the
     future → skip the job.
   - Otherwise (not suspended, or `SuspendedUntil` has passed) → proceed.
3. For each active entry, parse the `ScheduleDefinition` JSONC via `JOSYN.Commons.Schedule`
   and evaluate it against `DateTime.Now`.
4. For every entry that is due, call `SessionLauncher.LaunchSession()` with the job name
   and the `ArgumentRecordName` from that entry.

One launch per due entry per invocation. Two entries firing simultaneously produce two
independent session launches.

---

## Alternatives considered

### File-based schedule storage (one JSONC file per job)

Each job's schedule configuration stored as a `.jsonc` file on disk, in a convention-based
location under `BackendRoot\JobRepository\<JobName>\`. ADR-026 left this option open.

Rejected: the database is already the authoritative store for job registrations and session
state. A file-based schedule store would introduce a second store that operators must
synchronise with the DB, that cannot be queried transactionally, and that cannot be managed
through a future admin UI without separate file I/O logic. The inline-JSONC-in-DB approach
retains full ADR-026 schedule expressiveness while keeping all job configuration in one
governed location.

### A single flat table (no header/entry split)

One row per `(JobName, ArgumentRecordName)` with `Suspended` and `SuspendedUntil` repeated
on every entry row.

Rejected: suspension is a per-job concern, not per-entry. Storing it on every entry row
creates update anomalies — suspending a job requires updating every entry row for that job
atomically. A header row for the per-job state and child rows for per-entry state is the
correct normal form.

### Storing a parsed representation instead of raw JSONC

Pre-parse the `ScheduleDefinition` and store a normalised relational form (one row per rule,
typed columns for each rule type).

Rejected: it would require a new table per rule type, making the schema brittle against
ADR-026 language extensions. Storing JSONC verbatim delegates all rule-shape knowledge to
`JOSYN.Commons.Schedule`, keeps the DB schema stable, and lets the schedule language evolve
without schema migrations.

---

## Open questions

1. **`once`-rule consumed state** ✅ **Closed by ADR-029.** `josyn.FiredSlots`
   (introduced in ADR-029) serves as the consumed-state record. The PK
   `(JobName, ArgumentRecordName, SlotTime)` where `SlotTime` equals the `once` rule's
   `FireAt` value deduplicates the rule across all ticks. No separate table or column needed.

2. **Argument record definition** — `ArgumentRecordName` is an opaque string in this ADR.
   The definition of what an argument record is, how it is stored in the database (extending
   the job registry with 0-to-n named INI-payload rows per job), and how `TimeScheduler`
   resolves a name to a payload is deferred to a separate ADR. That ADR is a prerequisite
   for a fully operational `TimeScheduler`; this ADR can be implemented in parallel up to
   the point of the actual session launch.

3. **Write operations** — this ADR defines only the read interface (`GetAll`). Adding,
   updating, and removing `JobSchedule` records (via a CLI tool or admin UI) requires a
   write contract. Deferred until the management tooling need is concrete.

---

## Consequences

- `josyn-backend` gains a new NuGet package: `JOSYN.Backend.JobScheduleStore`.
- Two new tables are added to `josyn-backend/db/migrations/`: `V003__job_schedules.sql`
  (or the next available version number).
- `architecture/storage.md` — the Domains table gains a new row:
  `Job schedule store | JOSYN.Backend.JobScheduleStore | josyn.JobSchedules + josyn.JobScheduleEntries | Planned`.
- `TimeScheduler.exe` replaces its hardcoded `FindNextDueJobTypeName()` stub with a call to
  `IJobScheduleStore.GetAll()` and real evaluation logic.
- ADR-026 open question "file naming convention and storage location" is closed by this ADR:
  storage is in the database, inline per entry.

---

## Relation to other ADRs

- **ADR-007** (Storage Realm): this ADR adds a new domain to the storage realm under the
  existing governance rules.
- **ADR-007B-02** (JobRegistry): `JobSchedule.JobName` is a foreign key into the job
  registry. A `JobSchedule` cannot exist for a job that is not registered.
- **ADR-024** (Ticker): the Ticker fires `TimeScheduler.exe`; this ADR defines what
  `TimeScheduler` reads when it wakes up.
- **ADR-026** (Schedule Definition Language): the `ScheduleDefinition` column stores ADR-026
  JSONC text verbatim. `JOSYN.Commons.Schedule` parses it at evaluation time.
