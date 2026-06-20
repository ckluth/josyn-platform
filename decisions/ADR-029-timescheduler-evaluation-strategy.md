# ADR-029 — TimeScheduler Evaluation Strategy: Tolerance Window and Fired-Slot Log

**Date:** 2026-06-20
**Status:** Accepted

---

## Context

ADR-024 established `TimeScheduler.exe` as the orchestrator responsible for time-based job
launches. It is a ticker target: invoked externally by the Ticker (ADR-024) at a configurable
cadence — typically every 30 to 60 seconds — then it exits. It holds no state between
invocations.

ADR-026 defined the schedule definition language. ADR-027 defined the `JobScheduleStore`
that persists scheduling configuration per job. With those foundations in place, `TimeScheduler`
needed a concrete algorithm for answering: *"is this schedule entry due right now?"*

### The problem with exact-moment matching

The first implementation evaluated each entry against the exact current minute (`HH:mm`),
ignoring seconds. This is correct and sufficient as long as the Ticker fires reliably once
per minute. It breaks silently in two ways:

1. **Maintenance windows.** If the server is restarted or the Ticker is paused for any reason,
   every scheduled slot that fell inside the outage window is missed permanently. For a job
   that runs every few minutes this is inconsequential — the next occurrence is imminent. For
   a job that runs once a day or once a week, a missed slot is a business problem.

2. **Sub-minute double-fire.** If two ticks fall within the same clock minute (possible when
   the period is 30 s), both see the same `HH:mm` slot as due and launch the job twice.

### What is needed

A strategy that:
- **Fires exactly once** per scheduled slot, regardless of how many ticks fall near it.
- **Recovers** a slot that was missed during a short outage, up to a configurable time limit.
- **Does not spontaneously catch up** for slots that are older than that limit — a long outage
  should not produce a burst of stale launches on restart.
- Keeps the per-slot tolerance **configurable per schedule entry**, because the right tolerance
  for a 5-minute interval job is very different from the right tolerance for a weekly job.

---

## Vocabulary

| Term | Definition |
|------|-----------|
| **Tick** | One invocation of `TimeScheduler.exe`, triggered by the Ticker at its configured cadence. |
| **S** | The canonical scheduled fire time of a rule: the exact `YYYY-MM-DD HH:mm` moment at which a rule is defined to fire, derived purely from the schedule definition, independent of when any tick happens to run. |
| **T** | The tolerance window in minutes, configured per schedule entry: how many minutes after `S` a tick may still legitimately fire that slot. |

---

## Decision

### 1. Tolerance window per entry

Each `JobScheduleEntry` row carries a nullable `ToleranceMinutes` integer column.

- When set, it is the entry-specific T.
- When `NULL`, the platform default applies. The recommended platform default is **1 minute**,
  which is safe for a 30–60 s tick cadence: the window is small enough that only one or two
  ticks can ever fall inside it, and deduplication (§2) handles the rest.

A slot `S` is **within tolerance** for a tick arriving at `now` when:

```
S <= now <= S + T
```

Only the forward direction is used: a tick that arrives *before* S does not fire it early.

### 2. Latest-candidate simplification

For a given rule and a given tick at `now`, the algorithm identifies the **most recently
scheduled slot** `S` that falls within `[now − T, now]` — not all slots in the window.

This simplification is correct in practice because T is expected to be small relative to
the spacing between consecutive slots. When T is properly configured (see §5), at most one
slot falls in any window, and "latest" is identical to "all."

If T is misconfigured to be larger than the spacing between consecutive slots of an `interval`
rule, intermediate slots are silently dropped — only the latest is considered. This is a
deliberate trade-off: catching up a burst of identical interval-job runs after a maintenance
window is typically worse than skipping them. See §5 for the constraint.

### 3. Fired-slot log for deduplication

A new table `josyn.FiredSlots` records every slot that has been acted upon. Its deduplication
key is `(JobName, ArgumentRecordName, SlotTime)` where `SlotTime` is the canonical `S`, not
`now`. Multiple ticks that fall within `[S, S + T]` all resolve to the same `S` and therefore
hit the same log key.

Before launching a session, `TimeScheduler` writes the log entry. Only if the write succeeds
(i.e. the key did not already exist) does it proceed to launch. This is **at-most-once**
semantics: a failed launch within the tolerance window is not automatically retried.

The log is not a permanent audit trail. Rows are pruned on each invocation: any entry whose
`SlotTime` is older than `now − T − CleanupBuffer` is deleted. `CleanupBuffer` is a fixed
platform constant (recommended: 10 minutes) that provides a safety margin against clock drift
and overlapping invocations.

### 4. Processing algorithm

**On each tick:**

- Read `now`, truncated to minute precision.
- Prune the fired-slot log: delete rows where `SlotTime < now − T_max − CleanupBuffer`,
  where `T_max` is the largest `ToleranceMinutes` value across all active entries.
- Load all job schedules from `SqlJobScheduleStore`.

**Per schedule:**

- Skip the schedule entirely if it is suspended.

**Per schedule entry:**

- Resolve T: use the entry's `ToleranceMinutes` if set; otherwise use the platform default.
- Parse the entry's `ScheduleDefinition`.
- Find the latest slot `S` in `[now − T, now]` by stepping backward from `now`, one minute
  at a time, and asking the rule evaluator whether each candidate minute is a scheduled fire
  time. Stop at the first hit; this is `S`.
- If no `S` is found in the window: the entry is not due — skip.
- Look up `(JobName, ArgumentRecordName, S)` in `josyn.FiredSlots`.
- If the key exists: this slot was already handled by a previous tick — skip.
- If the key does not exist:
  - Write `(JobName, ArgumentRecordName, S, now)` to `josyn.FiredSlots`.
  - Resolve the argument record via `IJobRegistry.GetArgument`.
  - Launch the session via `SessionLauncher`.

### 5. Constraint: T must be less than the shortest `every` period

For `interval` rules, the spacing between consecutive slots equals the `every` duration.
If T ≥ `every`, two or more consecutive slots can fall within `[now − T, now]` simultaneously.
The latest-candidate simplification (§2) will then silently drop the older ones.

**Operators are responsible** for ensuring that T < `every` for every `interval` rule in an
entry's schedule definition. The schedule validator may emit a warning when it detects a
`ToleranceMinutes` value on an entry that also contains an `interval` rule with a shorter
period, but cannot enforce this constraint because `ToleranceMinutes` lives on the entry and
the interval period lives inside the `ScheduleDefinition` JSONC — they are in different layers.

For all other rule types (`fixed`, `monthly_date`, `nth_weekday`, `week_interval`, `once`)
there is at most one candidate slot per day, so any reasonable T is safe.

### 6. At-most-once semantics

The log entry is written **before** the session is launched. If the launch fails, the slot
is considered consumed: it will not be retried automatically within the tolerance window.

The rationale: automatic retry within T would require distinguishing a launch failure from
a normal "already fired" state, which requires an additional status column and re-evaluation
logic. That complexity belongs in a dedicated retry mechanism, not in the scheduler. An
operator who needs guaranteed execution can set T large enough that a manual re-trigger is
possible before the window closes.

---

## Schema

### `josyn.JobScheduleEntries` — extended

Add one nullable column:

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `ToleranceMinutes` | `INT` | Yes | Override T for this entry. `NULL` means use the platform default (1 minute). |

### New table: `josyn.FiredSlots`

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `JobName` | `NVARCHAR(200)` | No | Matches `josyn.JobSchedules.JobName`. |
| `ArgumentRecordName` | `NVARCHAR(200)` | No | Matches `josyn.JobScheduleEntries.ArgumentRecordName`. |
| `SlotTime` | `DATETIME2` | No | The canonical scheduled fire time S. |
| `FiredAt` | `DATETIME2` | No | The actual `now` of the tick that handled this slot. |

**Primary key:** `(JobName, ArgumentRecordName, SlotTime)`

Rows are pruned by `TimeScheduler` on each invocation; no separate cleanup job is needed.

---

## Consequences

- `josyn.JobScheduleEntries` gains a `ToleranceMinutes` column. Existing rows default to
  `NULL` (platform default applies). The bootstrap DDL and `schema.md` are updated.
- A new table `josyn.FiredSlots` is added to the bootstrap DDL and `schema.md`.
- `JOSYN.Backend.JobScheduleStore` — `IJobScheduleEntryRecord` and `JobScheduleEntryRecord`
  gain a `ToleranceMinutes` nullable integer property.
- `JOSYN.Commons.Schedule` — `ScheduleEvaluator.IsDue` signature is unchanged. The caller
  (`TimeScheduler`) now drives the tolerance window by iterating candidate minutes and calling
  the evaluator once per candidate.
- `JOSYN.Backend.TimeScheduler` — the `IsDue` method is replaced by the full tolerance+log
  algorithm. The fired-slot log write and prune operations are added.
- **At-most-once** delivery is the defined platform contract for scheduled job launches.
  Retry is out of scope for this ADR.

---

## Open Questions

### OQ1 — T_max for pruning ✅ Closed

**Decision:** use a fixed upper bound of **1440 minutes (1 day)** as the prune cutoff
ceiling rather than computing `MAX(ToleranceMinutes)` across active entries.

Rationale: an aggregation query per tick adds a round-trip for no meaningful benefit.
The only consequence of a fixed ceiling is that a handful of rows are retained slightly
longer than strictly necessary — harmless.

The prune cutoff is therefore:

```
SlotTime < now − 1440 minutes − CleanupBuffer (10 minutes)
```

### OQ2 — Validator cross-layer warning

Deferred. The constraint (T must be less than the shortest `every` period among `interval`
rules in the same entry) is precisely documented in §5. Implementing a warning requires the
validator to parse the `ScheduleDefinition` JSONC of every entry to inspect `interval`
rule periods — a non-trivial cross-layer check for one operator-misconfiguration edge case.
Add when it proves necessary in practice.

### OQ3 — `once`-rule consumed state ✅ Closed

**Decision:** `josyn.FiredSlots` is the consumed-state record for `once` rules.
The PK `(JobName, ArgumentRecordName, SlotTime)` where `SlotTime` is the `once` rule's
`FireAt` value naturally deduplicates the rule across all ticks without a separate table.
No additional mechanism is needed. The ADR-027 open question on this topic is closed
by back-reference to this ADR.

---

## Relation to Other ADRs

- **ADR-024** (Ticker): establishes the invocation cadence and single-instance guard that
  this strategy assumes.
- **ADR-026** (Schedule Definition Language): defines the rule types and the canonical `S`
  semantics that the evaluator implements.
- **ADR-027** (JobSchedule Store): defines `josyn.JobSchedules` and `josyn.JobScheduleEntries`;
  this ADR extends the latter with `ToleranceMinutes`.
