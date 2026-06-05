# JOSYN Storage Realm

The JOSYN Storage Realm is the single, owned, governed location for all durable
platform state. See [ADR-007](../decisions/ADR-007-storage-realm.md) for the
full rationale and decision record.

---

## Developer Setup

To bootstrap a fresh local development database, run one script:

```
josyn-backend/db/bootstrap-local-dev.sql
```

This creates the database, schema, login, and all current tables in one step.

For incremental updates to an **existing** dev DB (e.g. after pulling new migrations),
apply only the new `Vxxx__...sql` file(s) from `migrations/` in order.

---

## Technology Stack

| Concern           | Technology                              |
|-------------------|-----------------------------------------|
| Storage engine    | SQL Server                              |
| Access layer      | EF Core (SQL Server provider)           |
| Schema            | `josyn`                                 |
| Dev database name | `josyn-db-local`                        |
| Dev login         | `tu.josyn` / `josyn` (local dev only)   |

EF Core internals (entity types, DbContext) are always `internal sealed` — they
never appear in the public API of a storage package.

---

## Domains

| Domain                  | Package                          | Table                        | Status   |
|-------------------------|----------------------------------|------------------------------|----------|
| Session records         | `JOSYN.Backend.SessionStore`     | `josyn.SessionStore`         | Existing |
| Job registry            | `JOSYN.Backend.JobRegistry`      | `josyn.JobRegistry`          | Planned  |
| Error records           | `JOSYN.Backend.ErrorHandler`     | `josyn.ErrorStore`           | Existing |
| Configuration           | `JOSYN.Backend.ConfigStore`      | `josyn.ConfigStore`          | Existing |
| Infrastructure registry | *(TBD)*                          | *(TBD)*                      | Future   |

Each new domain must be recorded as an ADR or in the owning component's documentation
before implementation begins.

---

## Naming Conventions

### Public data contracts — `IxxxRecord`

All public interfaces representing a stored record follow the `IxxxRecord` convention:

```csharp
public interface IJobSessionRecord
{
    Guid   UID         { get; init; }
    string JobTypeName { get; init; }
    string Arguments   { get; init; }
    string Result      { get; init; }
}
```

Concrete types are `public sealed record` implementing the interface:

```csharp
public sealed record JobSessionRecord : IJobSessionRecord { ... }
```

### EF Core internals — always `internal sealed`

```csharp
internal sealed class SessionStoreEntity { ... }
internal sealed class SessionStoreDbContext : DbContext { ... }
```

### Migration scripts — `Vxxx__description.sql`

```
V001__session_store.sql
V002__job_registry.sql
```

Zero-padded three-digit sequence, double underscore separator, lowercase snake-case
description. One file per schema change. The sequence is append-only — existing files
are never modified.

---

## Migration Script Location

```
josyn-backend/db/
├── bootstrap-local-dev.sql          ← dev bootstrap: creates DB + schema + login
└── migrations/
    ├── V001__session_store.sql
    ├── V002__job_registry.sql
    └── ...
```

Scripts are applied manually. No migration runner tool is used.

---

## ConfigStore Key Catalogue

All well-known keys are defined as constants in `JOSYN.Backend.ConfigStore.ConfigKeys`.
Use the constant — never a raw string.

| Key | Type / Valid Values | Written by | Read by | Notes |
|-----|---------------------|------------|---------|-------|
| `RuntimeEnvironment` | `RuntimeEnvironment` enum name: `DEV`, `INT`, `PROD` | Installation / setup script | `JAPServer.GetEnvironment()` | One value per installation. See ADR-010. |
