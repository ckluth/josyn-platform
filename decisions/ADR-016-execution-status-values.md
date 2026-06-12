# ADR-016 — ExecutionStatus: Closed Value Set

**Date:** 2026-06-12
**Status:** Accepted

---

## Context

`IJobSessionRecord.ExecutionStatus` is a `string` column in `josyn.SessionStore`
(`VARCHAR(32) NOT NULL`). It tracks where a job session is in its lifecycle.

Currently the field is assigned exactly one value — `"pending"` — at session creation,
in two places:

- `JOSYN.Backend.SessionStarter` (`SessionStarter.cs`)
- `JOSYN.Jap.JAPServer` (`Host.HandleSessionStart`)

No transitions are implemented. Once a session is saved, the status never changes.
The set of valid values has never been defined, documented, or enforced.

---

## Decision

`ExecutionStatus` is a **closed string set**. The following eight values are the only
permitted values. No other value may be written to this column.

### Value definitions

| Value | Kind | Description |
|---|---|---|
| `pending` | transient | Session record written; JAPServer and job process not yet spawned. |
| `running` | transient | Both processes are active; job is executing. |
| `running-cancellation-requested` | transient | A cancellation signal has been issued; the job has not yet honoured it. |
| `finished-successfully` | terminal | Job completed and called `PutRawResult` with a success result. |
| `finished-with-errors` | terminal | Job ran to completion and called `PutRawResult` with a domain error result — technically clean exit, but the job itself determined something went wrong in its subject area. |
| `finished-faulted` | terminal | An unhandled exception bubbled out of the job entry point; job-host called `PutError`. The job never reached a deliberate outcome. |
| `finished-by-cancellation` | terminal | Job honoured the cancellation request and terminated. |
| `finished-rejected` | terminal | Session was accepted by the scheduler but rejected during pre-execution negotiation (see ADR-008). |
| `finished-abandoned` | terminal | Processes died without reporting any outcome; detected and written by an external watchdog. |

### State machine

```
                    ┌──────────────────────────────────────────────────────┐
                    │                                                      │
     ┌───────────── pending ──────────────────────┐                       │
     │                  │                          │                       │
     │                  ▼                          ▼                       │
     │              running ───────► running-cancellation-requested        │
     │             /   |   \                   │                           │
     ▼            ▼    ▼    ▼                  ▼                           ▼
finished-   finished- finished- finished-  finished-by-         finished-
rejected  successfully  with-  faulted    cancellation          abandoned
                        errors
```

- `finished-successfully` — `PutRawResult` called with a success result
- `finished-with-errors` — `PutRawResult` called with a domain error result
- `finished-faulted` — unhandled exception; `PutError` called by job-host

All `finished-*` values are terminal — a session in a terminal state is never updated again.

### Invariant: "terminal implies done"

Any query that needs to know whether a session is complete can use a single pattern:

```sql
WHERE ExecutionStatus LIKE 'finished-%'
```

### Column constraint

The column definition is `VARCHAR(32)`. All eight values fit within this limit
(longest: `running-cancellation-requested` at 30 characters).

No database `CHECK` constraint is added at this time; enforcement is at the application
layer. A future migration may add a `CHECK` constraint once the value set has been
stable across at least one release cycle.

---

## Rationale

1. **Clarity over flexibility.** A string field with no defined values is effectively
   an untyped slot. Defining the set makes every transition auditable and every query
   against the column predictable.

2. **`finished-*` prefix for all terminal states.** A single `LIKE 'finished-%'` predicate
   identifies any completed session regardless of outcome. Monitoring, cleanup, and
   reporting queries need no knowledge of individual terminal values.

3. **`running-cancellation-requested` as a sub-state of running.** The prefix encodes a
   valid-transition rule: cancellation can only be requested while a session is `running`.
   The name makes this constraint readable without consulting documentation.

4. **`finished-rejected` rather than `rejected`.** Rejection is a terminal outcome.
   Placing it in the `finished-*` family keeps the terminal-detection invariant uniform.

5. **`finished-faulted` vs `finished-with-errors`.** Both are error outcomes, but they
   are different in nature. `finished-faulted` means an unhandled exception escaped the
   job entry point — the job never reached a deliberate outcome; job-host called `PutError`.
   `finished-with-errors` means the job ran to completion, evaluated its subject area, and
   *decided* something was wrong — job-host called `PutRawResult` with a domain error result.
   A retry policy, alerting rule, or operator treats these differently: a fault may indicate
   a defect or infrastructure problem; a domain error is an expected, handled outcome.

6. **`finished-abandoned` for silent process death.** `finished-faulted` means the
   job *reported* an error through the protocol. `finished-abandoned` means neither
   outcome nor error was ever reported — the processes simply vanished. The distinction
   is material for diagnostics: one means the job knew it failed; the other means nobody
   knows what happened.

7. **String type retained.** An enum column would require a DDL migration for every new
   value. Keeping `VARCHAR(32)` with application-layer governance preserves the ability
   to introduce new values without a schema change during the platform's formative period.

---

## Consequences

- All code that writes `ExecutionStatus` must use only the eight values defined above.
  Magic string literals are replaced with constants (a `ExecutionStatusValues` static
  class or equivalent) in the implementing packages.
- Four values (`running-cancellation-requested`, `finished-by-cancellation`,
  `finished-rejected`, `finished-abandoned`) require features not yet built:
  cancellation support, accept/reject negotiation (ADR-008), and a session watchdog.
  The values are defined here so that future implementations have a governed target;
  no new ADR is needed solely to name them.
- The `pending → running` transition — currently missing — must be implemented when
  JAPServer begins execution. This is the first gap to close after this ADR.
