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

---

## What was done — continued (2026-06-20, session 2)

### ADR-029 implemented end-to-end ✅

**a. Schema** (`josyn-backend/db/`)
- `bootstrap-local-dev.sql`: `[ToleranceMinutes] INT NULL` added to `josyn.JobScheduleEntries`;
  new `josyn.FiredSlots` table (PK: `JobName + ArgumentRecordName + SlotTime`).
- `schema.md`: `josyn.JobScheduleEntries` updated (column + ADR reference corrected);
  new `josyn.FiredSlots` section added; FK map updated.

**b. `JOSYN.Backend.JobScheduleStore`** — rebuilt and repacked ✅
- `IJobScheduleEntryRecord` / `JobScheduleEntryRecord` / `JobScheduleEntryEntity` / `DbContext` mapping:
  `int? ToleranceMinutes` added throughout.
- `SqlJobScheduleStore.ToRecord()`: maps `ToleranceMinutes` from entity.
- New `Db/FiredSlotEntity.cs` — EF entity for `josyn.FiredSlots`.
- New `Db/FiredSlotStoreDbContext.cs` — dedicated EF context for the fired-slot log.
- New `Contracts/IFiredSlotStore.cs` — `TryInsert(...) → Result<bool>`, `Prune(cutoff) → Result`.
- New `SqlFiredSlotStore.cs` — uses `ExecuteSql` (FormattableString) with a `WHERE NOT EXISTS`
  guard for at-most-once insertion; `Prune` deletes by `SlotTime < cutoff`.

**c. `JOSYN.Backend.TimeScheduler`** — builds clean, 0 warnings ✅
- `IsDue()` stub removed; full ADR-029 algorithm in place:
  - `TruncateToMinute(DateTime)` — `now` is always minute-precision.
  - `PruneFiredSlots()` — called once per invocation before schedule processing.
  - `ProcessEntry()` — `FindLatestSlot` → `TryInsert` → launch; returns 1/0/-1.
  - `FindLatestSlot()` — step-backward loop through `[now − T, now]`.
  - `DefaultToleranceMinutes = 1`.

**d. `josyn-toolbox/deploy/deploy-maintainer.ps1`** ✅
- Step 5b comment updated: ADR-029 added to reference list.

**e. `josyn-toolbox/josyn-db-viewer/get-db-reports.ps1`** ✅
- `$entryTable` query: `ToleranceMinutes` added to SELECT.
- `$entryIndexByKey` index added (for FiredSlots → ScheduleEntry cross-links).
- `$firedSlotTable` query added.
- JobScheduleEntries overview + details: `ToleranceMinutes` column rendered.
- New `fired-slots.md` report section: overview table + per-slot detail blocks with
  cross-links to job-registry, argument-records, and job-schedule-entries.
- README index: `josyn.FiredSlots` row added.

---

## What is still missing / next steps

### 1. ~~Implement ADR-029 evaluation strategy~~ ✅ Done (session 2)

---

### 2. ~~`once`-rule consumed state — closed by ADR-029~~ ✅ Already done (ADR-027 OQ1 updated in session 1)

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

---

### 6. ~~`josyn-toolbox/deploy` — update ADR reference comment~~ ✅ Done (session 2)

---

### 7. ~~`josyn-toolbox/josyn-db-viewer` — add FiredSlots, update Entries~~ ✅ Done (session 2)
