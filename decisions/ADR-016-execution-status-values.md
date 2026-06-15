# ADR-016 — ExecutionStatus: Closed Value Set

**Date:** 2026-06-12
**Status:** Accepted

---

## Context

`IJobSessionRecord.ExecutionStatus` is a `string` column in `josyn.SessionStore`
(`VARCHAR(32) NOT NULL`). It tracks where a job session is in its lifecycle.

Currently the field is assigned exactly one value — `"preparing"` — at session creation
by `JOSYN.Jap.JAPServer` (`Host.Starter.cs`) inside the Turnstile start phase.

No transitions are implemented. Once a session is saved, the status never changes.
The set of valid values has never been defined, documented, or enforced.

---

## Decision

`ExecutionStatus` is a **closed string set**. The following eight values are the only
permitted values. No other value may be written to this column.

### Value definitions

| Value | Kind | Description |
|---|---|---|
| `preparing` | transient | Session record written; JAPServer is active; `job.exe` is launching, connecting pipes, and the accept/reject negotiation (ADR-008) is in progress. |
| `running` | transient | Both processes are active; job is executing. |
| `running-cancellation-requested` | transient | A cancellation signal has been issued; the job has not yet honoured it. |
| `finished-successfully` | terminal | Job ran to completion without error. `PutRawResult` was called if the job produces a result; void jobs complete successfully without it. |
| `finished-with-errors` | terminal | Job ran to completion and called `PutDomainError` — technically clean exit, but the job itself determined something went wrong in its subject area. |
| `finished-faulted` | terminal | An unhandled exception bubbled out of the job entry point; job-host called `PutError`. The job never reached a deliberate outcome. |
| `finished-by-cancellation` | terminal | Job honoured the cancellation request and terminated. |
| `finished-rejected` | terminal | The job rejected the session during accept/reject negotiation (ADR-008) — a concurrent instance is already running and the job's parallel execution policy forbids it. |
| `finished-abandoned` | terminal | Processes died without reporting any outcome; detected and written by an external watchdog. |

### State machine

```
                    ┌──────────────────────────────────────────────────────┐
                    │                                                      │
            preparing ──────────────────────────────────────────────────┤
           /         \                                                    │
          ▼           ▼                                                   │
       running    finished-rejected                                       │
      /   |   \                                                           │
     ▼    ▼    ▼    running-cancellation-requested                        │
  fin.  fin.  fin.          │                                             │
  succ  with  faulted       ▼                                             │
        err           finished-by-cancellation                            │
                                                                          │
                                                ──────────────────────► finished-   │
                                                                       abandoned ◄─┘
```

Prose form:

- `preparing → running` — job accepted the session (ADR-008)
- `preparing → finished-rejected` — job rejected the session, or negotiation timed out (ADR-008)
- `running → running-cancellation-requested` — cancellation signal issued
- `running → finished-successfully` — `PutRawResult` called
- `running → finished-with-errors` — `PutDomainError` called (ADR-018)
- `running → finished-faulted` — unhandled exception; `PutError` called by job-host
- `running-cancellation-requested → finished-by-cancellation` — job honoured cancellation
- any state → `finished-abandoned` — watchdog detects silent process death

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
   *decided* something was wrong — job-host called `PutDomainError` (ADR-018).
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
  Magic string literals are replaced with constants (an `ExecutionStatusValues` static
  class or equivalent) in the implementing packages.
- Four values (`running-cancellation-requested`, `finished-by-cancellation`,
  `finished-abandoned`, and `finished-with-errors`) require features not yet fully built:
  `finished-with-errors` requires the `PutDomainError` verb specified in ADR-018;
  cancellation support and a session watchdog remain future work.
  The values are defined here so that future implementations have a governed target.
- `finished-rejected` is reachable once ADR-008 is implemented.
