# ADR-002 — SessionStore

**Date:** 2026-06-03
**Status:** Accepted

---

## Context

`JOSYN.Backend.SessionStore` is the first real Category A component in `josyn-backend`,
as established by ADR-001. Every job execution requires a persistent session record: a unique
GUID, the job type name, the serialized arguments, and (after completion) the serialized result.
This data bridges the three independent OS processes involved in a session
(`SessionStarter` → `JAPServer` → `job.exe`).

The playground prototype (`josyn-playground/.../SessionStore`, formerly `josyn-sandbox`) proved the EF Core model against
the legacy SQL Server schema. This ADR records the decisions taken when promoting that
prototype to a production NuGet package.

---

## Decisions

### 1. SQL Server + EF Core

The legacy job system already uses SQL Server with the `josyn` schema and a `SessionStore`
table. EF Core (SQL Server provider) is the natural fit: it matches the existing schema,
supports retry-on-failure, and integrates cleanly with the `Result` pattern via try/catch.

No ORM abstraction layer is introduced. EF Core is used directly — one `DbContext`,
one `DbSet`, configured entirely in `SessionStoreDbContext`.

### 2. Constructor injection for the connection string

`SessionStore` receives its connection string as a constructor parameter:

```csharp
public sealed class SessionStore(string connectionString) : ISessionStore
```

The caller is responsible for supplying the string. This keeps `SessionStore` ignorant of
how configuration is managed — that concern belongs to `JOSYN.Backend.GlobalConfig`
(see ADR-003, planned).

### 3. `ISessionStore.GetSession` accepts `Guid`, not `string`

The sandbox version (now in `josyn-playground`) accepted `string sessionUid` and parsed it internally. The promoted
version accepts `Guid` directly — the parsing concern belongs at the boundary where
the raw string arrives (e.g., CLI argument parsing in `JAPServer`), not in the store.

### 4. `DbContext` and `SessionStoreEntity` are internal

`SessionStoreDbContext` and `SessionStoreEntity` are `internal sealed`. They are
implementation details of `SessionStore` and must not leak into the public API.
Only `ISessionStore`, `IJobSessionRecord`, and `JobSessionRecord` are public.
The `IJobSessionRecord` / `JobSessionRecord` naming follows the platform-wide
`IxxxRecord` convention established in ADR-007.

### 5. No Mock in this package

The mock implementation (`MockSessionStore`) is deferred to the phase when it is first
needed (Phase 3 of the PoC roadmap). It will live in a separate `JOSYN.Backend.SessionStore.Mock`
NuGet package, per the ADR-001 pattern. Production assemblies must never carry test or demo
dependencies.

---

## Consequences

- `JAPServer` and future backend executables consume `ISessionStore` without knowing which
  implementation is active at runtime
- The legacy SQL Server schema is reused without modification — zero migration risk
- Adding a second `ISessionStore` implementation (e.g., for a different DB engine) requires
  no changes to consumers
