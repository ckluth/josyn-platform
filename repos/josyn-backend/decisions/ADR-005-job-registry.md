# ADR-005 — JobRegistry

**Date:** 2026-06-04
**Status:** Accepted

---

## Context

The legacy job system has no explicit job registry. Knowledge of which jobs exist
and under what identity they run is stored implicitly in the company's proprietary
global configuration database. This creates several problems:

- There is no single, queryable source of truth for registered jobs within JOSYN
- Adding or modifying a job registration requires access to a proprietary external system
- `SessionStarter` has no place to look up job metadata (e.g. which technical user
  to impersonate when spawning a JAPServer instance)
- The planned `Ticker` (ADR-004) has no place to enumerate which jobs are schedulable

Making the job registry an explicit JOSYN concept — with a dedicated SQL table and a
stable interface — closes this gap.

---

## Decision

### 1. New Category A NuGet: `JOSYN.Backend.JobRegistry`

`JOSYN.Backend.JobRegistry` is introduced as a Category A NuGet in `josyn-backend`,
following the building block model established in ADR-001. It owns the
`IJobRegistration` record contract, the `IJobRegistry` interface, and the
`SqlJobRegistry` production implementation.

Layout:

```
josyn-backend-job-registry/
├── nuget.config
├── JOSYN.Backend.JobRegistry.slnx
├── .local-build/
│   ├── build.cmd
│   ├── clean.cmd
│   └── pack.cmd
└── JOSYN.Backend.JobRegistry/
    ├── IJobRegistration.cs
    ├── IJobRegistry.cs
    └── SqlJobRegistry.cs
```

### 2. `IJobRegistrationRecord` — the registration record

```csharp
public interface IJobRegistrationRecord
{
    string Name { get; }
    string TechnicalUserName { get; }
}
```

`Name` is the platform-wide unique identifier for the job — the key by which all
backend components refer to a job: in session records, in scheduling configuration,
and in the registry itself.

`TechnicalUserName` is the OS-level user identity under which the job's JAPServer
instance will be spawned (e.g. `"tu.demojob"`). This enables per-job process
isolation and auditing.

The `IxxxRecord` naming follows the platform-wide stored data contract convention
established in ADR-007.

### 3. `IJobRegistry` interface

```csharp
public interface IJobRegistry
{
    Result<IJobRegistrationRecord>               GetByName(string name);
    Result<IReadOnlyList<IJobRegistrationRecord>> GetAll();
}
```

Both methods return `Result<T>` — no thrown exceptions, consistent with the
platform-wide Result pattern.

### 4. `SqlJobRegistry` — EF Core implementation

Registrations are stored in a new table `josyn.JobRegistrations` in the existing
SQL Server database — the same database and schema already used by `SessionStore`.

Constructor injection for the connection string follows the `SessionStore` precedent
(ADR-002, Decision 2):

```csharp
public sealed class SqlJobRegistry(string connectionString) : IJobRegistry
```

EF Core internals (`JobRegistryDbContext`, `JobRegistrationEntity`) are `internal
sealed` — no implementation detail leaks through the public API.

Per-method `DbContext` instantiation follows the `SessionStore` pattern (ADR-006).

The `IxxxRecord` naming convention (ADR-007) applies: the public data type is
`IJobRegistrationRecord` / `JobRegistrationRecord`.

### 5. Future legacy adapter path — noted, not designed

The company's proprietary configuration database is the current implicit registry in
the legacy system. An `IJobRegistry` implementation backed by that database is a
valid migration path during a parallel-existence phase. This is acknowledged here;
the adapter is not designed or implemented until the migration need is concrete. No
inner source abstraction or composite pattern is introduced at this time.

---

## Consequences

- `Ticker` and `SessionStarter` have a stable `IJobRegistry` contract to consume when
  scheduling and session-start logic is implemented
- `IJobRegistration.Name` becomes the shared key across `JobRegistry`, `SessionStore`
  (job name field in session records), and the future `JobRepository`
- Adding a legacy adapter requires only a new `IJobRegistry` implementation — no
  changes to consumers
- The EF Core migration for `josyn.JobRegistrations` must be applied to the shared
  SQL Server database before the package can be used in production
