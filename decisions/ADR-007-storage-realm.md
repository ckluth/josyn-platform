# ADR-007 — The JOSYN Storage Realm

**Date:** 2026-06-04
**Status:** Accepted

---

## Context

The legacy job system has no owned, governed storage concept. Durable state is
scattered across a SQL table, convention-based file-system locations, and the company
configuration manager. There is no single place that answers "what does this system
know and remember?" and no model for how schema changes are applied.

JOSYN must not inherit this disorder. As the platform grows — session records, job
registrations, error records, infrastructure configuration — an explicit storage model
is required before more domains are added.

---

## Decision

### 1. The JOSYN Storage Realm

JOSYN has a single, owned, governed storage space: the **JOSYN Storage Realm**.
It is the authoritative location for all durable platform state.

Current and anticipated domains:

| Domain                  | Table                        | Status   |
|-------------------------|------------------------------|----------|
| Session records         | `josyn.SessionStore`         | Existing |
| Job registrations       | `josyn.JobRegistrations`     | Planned  |
| Error records           | *(table TBD)*                | Future   |
| Infrastructure registry | *(table TBD)*                | Future   |

Each new domain requires a decision recorded either as an ADR or in the owning
component's documentation. The table list is append-only — existing tables are
never dropped or repurposed.

### 2. Technology: SQL Server + EF Core, behind a per-domain abstraction

SQL Server is the storage technology. EF Core (SQL Server provider) is the access
mechanism. Neither leaks into the public API of any storage package.

The abstraction is the per-domain interface (`ISessionStore`, `IJobRegistry`, etc.).
Consumers depend on the interface only — they are unaware of SQL Server, EF Core, or
schema details. A different storage technology is substitutable at the composition
root without touching consumers.

No single umbrella `IStore` interface is introduced. Per-domain interfaces are the
correct granularity: consumers need only the operations relevant to their domain.

### 3. `IxxxRecord` — naming convention for stored data contracts

All public interfaces representing a stored data record follow the `IxxxRecord`
convention:

```csharp
public interface IJobSessionRecord { ... }
public interface IJobRegistrationRecord { ... }
```

Concrete implementations are `public sealed record` types:

```csharp
public sealed record JobSessionRecord(...) : IJobSessionRecord { ... }
```

EF Core internals — entity types (`xxxEntity`) and context types (`xxxDbContext`) —
are always `internal sealed` and never part of the public API.

### 4. Align existing types — retroactive rename

`IJobSession` and `JobSession` in `JOSYN.Backend.SessionStore` are renamed to
`IJobSessionRecord` and `JobSessionRecord`. All consumers are updated in the same
session. The package version is bumped to mark the breaking change.

`IJobRegistration` and `JobRegistration` in the planned `JOSYN.Backend.JobRegistry`
(ADR-005) are renamed to `IJobRegistrationRecord` and `JobRegistrationRecord` before
implementation begins. ADR-005 is updated accordingly.

### 5. DDL scripts — location and naming convention

SQL DDL scripts live in `josyn-backend/db/`. This is the single owning repo for all
current storage domains.

Structure:

```
josyn-backend/db/
├── bootstrap-local-dev.sql          ← creates DB, schema, dev login (dev only)
└── migrations/
    ├── V001__session_store.sql      ← josyn.SessionStore
    ├── V002__job_registrations.sql  ← josyn.JobRegistrations
    └── ...                          ← one file per schema change, append-only
```

Scripts are applied manually — no migration runner tool is required. The
`migrations/` folder is the ordered, append-only history of all schema changes.

The `josyn-sandbox/db/create-database.sql` is the origin of `bootstrap-local-dev.sql`
and `migrations/V001__session_store.sql`. The sandbox file is superseded by these.

### 6. Architecture documentation

`architecture/storage.md` is the canonical reference for the JOSYN Storage Realm:
schema overview, domain table, technology stack, naming conventions, script location,
and developer setup instructions. It is the first document a developer reads when
setting up a local database or adding a new storage domain.

---

## Consequences

- All durable platform state has a single, documented home
- `IxxxRecord` is the platform-wide convention; existing types are aligned now, while
  the platform is young and the breaking change is cheap
- Storage technology is substitutable at the composition root without touching consumers
- DDL history is auditable: every schema change is a versioned SQL file in
  `josyn-backend/db/migrations/`
- A new developer can bootstrap a local database by reading `architecture/storage.md`
  and running two scripts
- The `josyn-sandbox/db/` SQL script is superseded and no longer the reference source
