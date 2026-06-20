# Plan — Time-Based Scheduling: From ADR to Working Orchestrator

**Created:** 2026-06-19
**Session:** ADR-027 + ADR-028 authoring and initial implementation

---

## What was done in this session

### Decisions (new ADRs)

| ADR | Title | Status |
|-----|-------|--------|
| ADR-027 | JobSchedule Store — scheduling configuration per job | Draft |
| ADR-028 | ArgumentRecord — named argument payloads in the job registry | Draft |

**ADR-026 open question closed:** storage location for `ScheduleDefinition` is now decided —
inline JSONC in `josyn.JobScheduleEntries`, not a file on disk.

### Schema

- `josyn-backend/db/bootstrap-local-dev.sql` — three new tables added:
  - `josyn.ArgumentRecords` — named INI payloads per job (composite PK: JobName + Name)
  - `josyn.JobSchedules` — per-job scheduling header with suspension state
  - `josyn.JobScheduleEntries` — one row per schedule entry (composite PK: JobName + ArgumentRecordName)
- `josyn-backend/db/schema.md` — new human-readable schema reference (mirrors bootstrap SQL)
- `ROADMAP.md` — schema migration strategy added to "Parked" section

### New NuGet package

**`JOSYN.Backend.JobScheduleStore`** (`josyn-backend/josyn-backend-job-schedule-store/`)
- `IJobScheduleStore` / `SqlJobScheduleStore` — `GetAll()` loads all schedules + entries
- `IJobScheduleRecord` / `JobScheduleRecord`
- `IJobScheduleEntryRecord` / `JobScheduleEntryRecord`
- Built and packed to local feed ✅

### Extended NuGet package

**`JOSYN.Backend.JobRegistry`** extended per ADR-028:
- `IArgumentRecord` / `ArgumentRecord` — new public contract
- `IJobRegistrationRecord` extended with `ArgumentRecords` list
- `IJobRegistry.GetArgument(jobName, argumentName)` — new targeted hot path
- EF Core: `ArgumentRecordEntity`, navigation on `JobRegistrationEntity`, DbContext updated
- Rebuilt and repacked ✅

### TimeScheduler — stub replaced

`TimeScheduler/Program.cs` now:
- Loads all job schedules via `SqlJobScheduleStore.GetAll()`
- Evaluates per-job suspension (flag + `SuspendedUntil` auto-lift) — **real logic**
- Resolves argument payloads via `SqlJobRegistry.GetArgument()` — **real logic**
- Launches one session per due entry — **real logic**
- `IsDue()` — **stub**, always returns `true` — clearly marked TODO

### Tooling

- `josyn-backend/.local-build/pack.cmd` — `josyn-backend-job-schedule-store` added to pack sequence
- `josyn-toolbox/deploy/deploy-maintainer.ps1` — TimeScheduler label updated (no longer "(stub)")

---

## What was done — continued (2026-06-20)

### `ScheduleEvaluator` — implemented ✅

`JOSYN.Commons.Schedule` — two new files:
- `Evaluation/ScheduleEvaluator.cs` — entry point `bool IsDue(ScheduleDefinition, DateTime)`,
  exclusion collection, active-window check (including ADR-026 annual year-boundary wrap)
- `Evaluation/ScheduleEvaluator.Rules.cs` — all six rule evaluators: `fixed`, `interval`,
  `monthly_date` (numeric / last / last_business), `nth_weekday` (month / quarter / year;
  numeric / last / last-N ordinals), `week_interval`, `once`

`JOSYN.Commons.Schedule.Test` — `ScheduleEvaluatorTests.cs`: 40 new tests.
All 140 tests in the suite pass. Package repacked to local feed.

`JOSYN.Backend.TimeScheduler` — `IsDue()` stub replaced: parse `entry.ScheduleDefinition`
via `ScheduleParser.Parse()`, then call `ScheduleEvaluator.IsDue()`. Builds clean.

### ADR-029 — authored ✅

`decisions/ADR-029-timescheduler-evaluation-strategy.md` (Draft).

Supersedes the exact-minute-match approach with a tolerance window + fired-slot log strategy.
Key decisions:
- **T** (tolerance, per `JobScheduleEntry`, nullable — default 1 minute)
- **Fired-slot log** (`josyn.FiredSlots`) deduplicates across ticks; PK = `(JobName, ArgumentRecordName, SlotTime)`
- **Latest-candidate-only** simplification: step backward from `now`, fire the first hit in `[now − T, now]`
- **At-most-once** semantics: log entry written before launch; failed launch is not retried within T
- **Constraint**: T must be < shortest `every` period of any `interval` rule in the same entry

---

## What is still missing / next steps

### 1. Implement ADR-029 evaluation strategy ← **natural continuation point**

The `ScheduleEvaluator` and `TimeScheduler` currently use the old exact-match approach.
ADR-029 must now be implemented end-to-end:

**a. Schema** (`josyn-backend/db/`)
- Add `ToleranceMinutes INT NULL` to `josyn.JobScheduleEntries` in `bootstrap-local-dev.sql`
- Add `josyn.FiredSlots` table to `bootstrap-local-dev.sql`
- Update `schema.md` to reflect both changes

**b. `JOSYN.Backend.JobScheduleStore`** (`josyn-backend/josyn-backend-job-schedule-store/`)
- Add `ToleranceMinutes int?` to `IJobScheduleEntryRecord` and `JobScheduleEntryRecord`
- Add `IFiredSlotStore` / `SqlFiredSlotStore`:
  - `TryInsert(jobName, argumentRecordName, slotTime) → bool` — inserts if not exists; returns false if duplicate
  - `Prune(cutoff) → void` — deletes rows where `SlotTime < cutoff`
- Rebuild and repack

**c. `JOSYN.Backend.TimeScheduler`** (`josyn-backend/josyn-backend-time-scheduler/`)
- Replace current `IsDue()` one-liner with the full ADR-029 algorithm:
  - Resolve T for the entry
  - Step backward from `now` in 1-minute increments through `[now − T, now]`
  - Call `ScheduleEvaluator.IsDue()` per candidate until a hit is found → this is S
  - Call `SqlFiredSlotStore.TryInsert(jobName, argName, S)`
  - Launch only if `TryInsert` returns true
- Add prune call at the top of each invocation

---

### 2. `once`-rule consumed state — closed by ADR-029

ADR-029 open question 3 resolves this: `josyn.FiredSlots` is already the consumed-state
record for `once` rules. No separate table or ADR needed. Close ADR-027 open question when
ADR-029 is accepted.

---

### 3. Argument record — import tooling

Existing `*.ini` files from `JobRepository\<JobName>\local-arguments\` need to be importable
into `josyn.ArgumentRecords`. A `josyn-toolbox` command (`import-arguments`?) is anticipated.

**For now:** insert manually via SQL or bootstrap seed data.

---

### 4. Write operations for JobSchedule

Adding, updating, suspending, and removing `JobSchedule` + `JobScheduleEntry` rows has no
governed interface yet. Currently done via direct SQL.

**When needed:** extend `JOSYN.Backend.JobScheduleStore` with a write contract, or build a
CLI sub-command.

---

### 5. DB — re-bootstrap required

Any developer who had the old `bootstrap-local-dev.sql` applied must drop and re-run it to
pick up the `ToleranceMinutes` column and the `josyn.FiredSlots` table.
PoC phase — drop-and-recreate is the agreed strategy.
