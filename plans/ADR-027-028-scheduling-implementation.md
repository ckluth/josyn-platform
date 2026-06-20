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

## What is still missing / deferred

### 1. `ScheduleEvaluator` — the `IsDue()` implementation ← **natural continuation point**

The most important remaining piece. Lives in `JOSYN.Commons.Schedule`.

Needs to evaluate `DateTime.Now` against a parsed `ScheduleDefinition` and return `true`
if any rule is currently satisfied (and no `exclude` rule blocks it).

Must handle all 7 rule types:

| Rule type | Key complexity |
|-----------|----------------|
| `interval` | Sliding window: `start + N × every` within `[start, end]`, correct day match |
| `fixed` | Exact `HH:mm` match on correct day |
| `nth_weekday` | Nth occurrence of weekday in month/quarter/year; `"last"` / `"last-N"` variants |
| `monthly_date` | Calendar day; `"last"` and `"last_business"` (scan backward, skip weekends + excludes) |
| `week_interval` | Phase-relative: `floor((today − anchor) / 7) mod everyWeeks == 0` |
| `once` | Exact `YYYY-MM-DD HH:mm` match; consumed-state persistence TBD |
| `exclude` | Blocklist applied after all other rules; always wins |

Plus `activeFrom` / `activeUntil` window modifiers on every bounded rule.

**Suggested approach:** a new `ScheduleEvaluator` static class in `JOSYN.Commons.Schedule`,
with a single entry point `bool IsDue(ScheduleDefinition definition, DateTime now)`.
`TimeScheduler.IsDue()` stub becomes a one-liner calling it.

**ADR needed?** Probably not — it is a pure implementation of ADR-026 semantics.
A plan file (like this one) is sufficient.

---

### 2. `once`-rule consumed state

Where is the "fired" flag persisted for a `once` rule? ADR-026 and ADR-027 both defer this.

Options:
- A sidecar table `josyn.OnceRuleFired (JobName, ArgumentRecordName, OnceDatetime)`
- A `ConsumedAt` column on `josyn.JobScheduleEntries` (nullable — only used by `once` entries)
- Written back into the `ScheduleDefinition` JSONC (mutating stored content — not clean)

**Recommendation when revisited:** sidecar table. Clean separation, no schema mutation,
queryable. Decide in a new ADR-027B-01.

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
get the three new tables. PoC phase — drop-and-recreate is the agreed strategy.

---

## Natural continuation point

**Start with the `ScheduleEvaluator`** in `JOSYN.Commons.Schedule`.

Order of implementation within the evaluator:
1. `activeFrom` / `activeUntil` window check (shared by all bounded rules)
2. `exclude` rule — build the date blocklist first (needed by `last_business`)
3. `fixed` — simplest rule, good starting point
4. `interval`
5. `monthly_date` (including `last_business`)
6. `nth_weekday`
7. `week_interval`
8. `once` (defer consumed-state; fire every tick until consumed-state is implemented)

Once the evaluator exists, replace the `IsDue()` stub in `TimeScheduler` with the real call.
At that point, the end-to-end time-based session launch is complete.
