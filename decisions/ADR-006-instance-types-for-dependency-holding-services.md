# ADR-006 — Instance Types for Dependency-Holding Services

**Date:** 2026-06-03
**Status:** Accepted

---

## Context

The coding standard "static wins (when in doubt)" sets `static class` as the default.
The tiebreaker is intentionally strong: if there is no clear reason to use an instance type,
the static form wins.

A recurring question arises when a type needs to store a dependency — typically a connection
string, a file path, or a configuration value — that is injected once at construction and
reused across all method calls. `SessionStore` is the canonical example:

```csharp
public sealed class SessionStore(string connectionString) : ISessionStore
{
    public Result SaveNewSession(IJobSession session) { ... }
    public Result<IJobSession> GetSession(string uid) { ... }
    public Result UpdateSession(IJobSession session) { ... }
}
```

At first glance this appears to violate "static wins" because the methods could, in principle,
accept `connectionString` as a parameter. The question this ADR answers: is that reading correct,
or is the instance form the right tool here?

---

## Decision

An instance type is justified — and the "static wins" tiebreaker does not apply — when **all
three of the following conditions are met**:

1. **Encapsulated dependency.** The type stores a value (connection string, path, endpoint,
   credential) that is injected once and is logically fixed for the lifetime of the object.
   Passing it on every call would be mechanical noise, not meaningful parameterisation.

2. **Natural identity.** Two instances with different dependencies are meaningfully distinct
   objects. A `SessionStore` pointing at database A is not interchangeable with one pointing
   at database B.

3. **Interface contract.** The type implements an interface, making the instance form a
   prerequisite for polymorphism — the canonical reason to choose `class` over `static class`
   in JOSYN.

When all three conditions are present, the instance form is the correct and intentional choice.
It is not a concession or an exception to the static-first rule — the rule itself lists
"mutable state that needs encapsulation" and "genuine polymorphism need" as the triggers for
choosing an instance type. This ADR makes that reasoning explicit and named.

### What does not qualify

A type that merely calls an external resource without storing any dependency does not meet
condition 1 and should remain `static`. Accepting a connection string as a method parameter
is the correct form in that case.

A type that stores state but exposes no interface does not meet condition 3. Revisit whether
the interface is missing or whether the static form is actually correct.

### DbContext lifetime — not a performance concern

Each method in `SessionStore` opens its own `DbContext`:

```csharp
using var ctx = new SessionStoreDbContext(_connectionString);
```

This is not an overhead or pooling concern — it is the correct EF Core pattern. `DbContext`
is designed as a short-lived unit-of-work object: one instance per operation, disposed
immediately after. Connection pooling happens at the ADO.NET / `SqlClient` layer and is
unaffected by how frequently `DbContext` is created and disposed. Sharing a single
`DbContext` across calls would introduce stale change-tracker state and subtle concurrency
bugs. The per-method form is intentional and correct.

---

## Consequences

- Agents and reviewers have a named reference (ADR-006) for resolving the static-vs-instance
  question when a dependency-holding service is introduced, without re-litigating the reasoning
  each time.
- The three-condition test provides a concrete checklist. A type that cannot satisfy all three
  conditions reverts to the static default.
- The decision does not weaken the "static wins" rule — it sharpens it by making the boundary
  explicit.
