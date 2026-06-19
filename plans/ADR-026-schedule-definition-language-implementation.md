# ADR-026 Implementation Plan — Schedule Definition Language

**Created:** 2026-06-19  
**Last updated:** 2026-06-19  
**ADR:** `josyn-platform/decisions/ADR-026-schedule-definition-language.md`

---

## Progress snapshot

| Phase | Title | Status |
|-------|-------|--------|
| 1 | New package scaffold — `JOSYN.Commons.Schedule` | ✅ Done |
| 2 | Value types | ✅ Done |
| 3 | Rule type hierarchy | ✅ Done |
| 4 | INI parser | ✅ Done |
| 5 | INI serializer | ✅ Done |
| 6 | Semantic validator | ✅ Done |
| 7 | Tests | ✅ Done |
| 8 | Documentation & docs-index | ✅ Done |

---

## Context

ADR-026 defines the schedule definition language consumed by `TimeScheduler.exe`.
This plan implements the language as a new `josyn-commons` package.
The package is domain-agnostic (no jobs, sessions, IPC) and meets all three commons
admission criteria.

---

## Phase 1 — New package scaffold

**New sub-folder:** `josyn-commons/josyn-commons-schedule/`  
**Package:** `JOSYN.Commons.Schedule`  
**Pattern:** B (new sub-solution inside existing multi-solution repo)

Deliverables:
- `JOSYN.Commons.Schedule.slnx` with main + test projects
- `JOSYN.Commons.Schedule.csproj` (net10.0, Nullable=enable, ResultPattern ref)
- `JOSYN.Commons.Schedule.Test.csproj`
- `nuget.config` pointing to `../../../local-packages/`
- `.local-build/build.cmd`, `test.cmd`, `pack.cmd`, `clean.cmd`

---

## Phase 2 — Value types

Files under `JOSYN.Commons.Schedule/Values/`:

| File | Type | Notes |
|------|------|-------|
| `Period.cs` | `enum Period` | `Month`, `Quarter`, `Year` |
| `DurationUnit.cs` | `enum DurationUnit` | `Minutes`, `Hours` |
| `Duration.cs` | `record Duration(int Amount, DurationUnit Unit)` | e.g. `30m`, `2h` |
| `Ordinal.cs` | `abstract record Ordinal` + `Numeric(int N)`, `Last`, `LastMinus(int Offset)` | `nth` field values |
| `MonthlyDay.cs` | `abstract record MonthlyDay` + `Numeric(int N)`, `Last`, `LastBusiness` | `day` field values |
| `DaySet.cs` | `record DaySet(IReadOnlySet<DayOfWeek> Days)` | Named factories: `Weekdays`, `Weekend`, `Daily` |
| `MonthSet.cs` | `record MonthSet(IReadOnlySet<int> Months)` | Months 1–12; named factory: `All` |
| `DateBound.cs` | `record DateBound` — mutually exclusive: `DateOnly? FullDate` / `(int Month, int Day)? Annual` | `active_from` / `active_until` values |
| `DateRange.cs` | `record DateRange(DateOnly Start, DateOnly End)` | `Start == End` = single date |

---

## Phase 3 — Rule type hierarchy

Files under `JOSYN.Commons.Schedule/Rules/`:

```
ScheduleRule.cs          ← abstract record ScheduleRule (DU root)
                            abstract record BoundedRule : ScheduleRule
                              (ActiveFrom DateBound?, ActiveUntil DateBound?)
IntervalRule.cs          ← sealed record : BoundedRule  (Days, Start, End, Every)
FixedRule.cs             ← sealed record : BoundedRule  (Days, Times)
NthWeekdayRule.cs        ← sealed record : BoundedRule  (Weekday, Nth, Period, Times)
MonthlyDateRule.cs       ← sealed record : BoundedRule  (Day, Times, Months?)
WeekIntervalRule.cs      ← sealed record : BoundedRule  (Days, Times, Every int, Anchor)
OnceRule.cs              ← sealed record : BoundedRule  (DateTime)
ExcludeRule.cs           ← sealed record : ScheduleRule (Dates) — no active window
```

Top-level container in `ScheduleDefinition.cs`:

```csharp
public sealed record ScheduleDefinition(IReadOnlyList<ScheduleRule> Rules);
```

---

## Phase 4 — INI parser

**`Parsing/ScheduleParser.cs`** — public static class  
**`Parsing/IniBlock.cs`** — internal; raw `Dictionary<string, string>` per block  
**`Parsing/ValueParsers.cs`** — internal; one static method per value type

Pipeline inside `ScheduleParser.Parse(string text) → Result<ScheduleDefinition>`:

1. Strip `#` / `;` comments, apply multi-line continuation (leading-whitespace lines join the preceding line).
2. Split into blocks at blank lines.
3. Each block → `IniBlock` (key-value dict).
4. Read `type` key → dispatch to per-rule factory.
5. Each factory calls `ValueParsers` for each field → returns `Result<TRule>`.
6. Accumulate all block errors; return combined failure if any block failed.

**Contract:**
```csharp
public interface IScheduleParser   // Contracts/IScheduleParser.cs
{
    static abstract Result<ScheduleDefinition> Parse(string text);
}
```

---

## Phase 5 — INI serializer

**`Parsing/ScheduleSerializer.cs`** — public static class  
`Serialize(ScheduleDefinition definition) → Result<string>`

Produces canonical INI: blocks separated by blank lines, keys in definition order,
multi-value fields (times, dates) use INI continuation indent for long lines.

**Contract:** added to `IScheduleParser` or a separate `IScheduleSerializer`.

---

## Phase 6 — Semantic validator

**`Validation/ValidationSeverity.cs`** — `enum ValidationSeverity { Error, Warning }`  
**`Validation/ValidationIssue.cs`** — `record ValidationIssue(ValidationSeverity Severity, string Message, int? RuleIndex)`  
**`Validation/ScheduleValidator.cs`** — public static class

`Validate(ScheduleDefinition definition) → IReadOnlyList<ValidationIssue>`
(Never fails structurally — always returns a list, possibly empty.)

**Error conditions:**

| Rule type | Condition |
|-----------|-----------|
| `interval` | `start >= end` |
| `interval` | `every` larger than the window — no fire slot possible |
| `week_interval` | `anchor` weekday not in `days` set |
| `exclude` | range where `start > end` |
| Any bounded rule | Both `active_from` / `active_until` are full dates and `from > until` |

**Warning conditions:**

| Rule type | Condition |
|-----------|-----------|
| `monthly_date` | Numeric `day` that some months in the active `months` set can never reach (e.g. `day=31` + `months=feb`) |
| `nth_weekday` | `nth=5` with `period=month` — most months have no 5th occurrence |
| `once` | `datetime` is in the past |
| Any | Duplicate rules (same type + same field values) |

**Contract:**
```csharp
public interface IScheduleValidator   // Contracts/IScheduleValidator.cs
{
    static abstract IReadOnlyList<ValidationIssue> Validate(ScheduleDefinition definition);
}
```

---

## Phase 7 — Tests

**`JOSYN.Commons.Schedule.Test/`**

Coverage areas:

- `DaySet` / `MonthSet` expression parsing (ranges, comma lists, keywords)
- `ValueParsers` — each value type: valid inputs, invalid inputs
- `ScheduleParser.Parse` — one happy-path test per rule type; error aggregation
- `ScheduleParser.Parse` — multi-line continuation; comment stripping
- `ScheduleSerializer` — round-trip: parse → serialize → parse produces equal definition
- `ScheduleValidator` — one test per Error condition; one test per Warning condition

---

## Phase 8 — Documentation & docs-index

- Update `josyn-platform/repos/josyn-commons.md` — add the new package row.
- Update `josyn-platform/docs/docs-index.json` — add entries for the new package.
- Update `josyn-platform/architecture/repo-structure-conventions.md` — correct the
  `josyn-commons` Pattern column (currently says "A (future)"; now confirmed Pattern B).
