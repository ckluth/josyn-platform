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
| 4 | JSON parser | 🔄 Rewrite (was INI) |
| 5 | JSON serializer | 🔄 Rewrite (was INI) |
| 6 | Semantic validator | ✅ Done |
| 7 | Tests | 🔄 Rewrite (was INI) |
| 8 | Documentation & docs-index | 🔄 Update |

---

## Context

ADR-026 defines the schedule definition language consumed by `TimeScheduler.exe`.
This plan implements the language as a new `josyn-commons` package.
The package is domain-agnostic (no jobs, sessions, IPC) and meets all three commons
admission criteria.

The format was revised from INI to JSONC after the initial implementation. The value
types (Phase 2) and rule hierarchy (Phase 3) are unchanged. The parser, serializer,
and tests require a full rewrite. The semantic validator requires no structural changes
but field-name references (`active_from`/`active_until` → `activeFrom`/`activeUntil`,
`time` → `times`) must be updated.

---

## Phase 1 — New package scaffold

**Status: done — no changes needed.**

---

## Phase 2 — Value types

**Status: done — no changes needed.**

The domain types (`DaySet`, `MonthSet`, `Duration`, `Ordinal`, `MonthlyDay`, `DateBound`,
`DateRange`) map cleanly to JSON. No structural changes required.

Note: `DaySet` range-expansion logic (`mon..fri`) and `MonthSet` range-expansion are no
longer needed in parsing — JSON arrays are expanded by the JSON layer. The named-shorthand
factories (`Weekdays`, `Weekend`, `Daily`) remain useful.

---

## Phase 3 — Rule type hierarchy

**Status: done — no changes needed.**

Field name changes in the rule records:

| Old (INI) | New (JSON) | Record |
|-----------|-----------|--------|
| `Times` (singular) | `Times` (already a list) | `FixedRule`, `NthWeekdayRule`, `WeekIntervalRule` |
| `Every` (int, weeks) | `EveryWeeks` | `WeekIntervalRule` |
| `ActiveFrom` / `ActiveUntil` | unchanged (camelCase was already C# naming) | `BoundedRule` |

`WeekIntervalRule.Every` (int) should be renamed to `WeekIntervalRule.EveryWeeks` to match
the JSON key name and remove the naming ambiguity with `IntervalRule.Every` (Duration).

---

## Phase 4 — JSON parser

**Replaces the INI parser. Delete:**
- `Parsing/IniBlock.cs`
- `Parsing/ScheduleParser.Preprocess.cs`

**Rewrite:**
- `Parsing/ScheduleParser.cs` — public static class
- `Parsing/ScheduleParser.Rules.cs` — per-rule-type JSON node → record factories

**Pipeline inside `ScheduleParser.Parse(string text) → Result<ScheduleDefinition>`:**

1. `JsonDocument.Parse(text, new JsonDocumentOptions { CommentHandling = JsonCommentHandling.Skip })`
2. Assert root element is a JSON array; fail with a clear message if not.
3. Iterate array elements → for each object, read `"type"` property → dispatch to
   per-rule factory in `ScheduleParser.Rules.cs`.
4. Each factory reads its required and optional properties from the `JsonElement` and
   calls `ValueParsers` for leaf string values.
5. Accumulate all element errors; return combined failure if any element failed.

**`ValueParsers.cs` — trim to leaf string parsers only. Remove:**
- `ParseDaySet` range/comma-list logic — replaced by array enumeration in the rule factory
- `ParseMonthSet` range/comma-list logic — same
- `ParseTimes` (comma-split) — replaced by JSON array enumeration
- `ParseDateRanges` (comma-split + `..`) — replaced by JSON array + range object handling

**Keep (unchanged semantics):**
- `ParseDaySet(string raw)` — now only handles single abbreviation or shorthand keyword
- `ParseWeekday(string raw)` — unchanged
- `ParseTimeOnly(string raw)` — unchanged
- `ParseDuration(string raw)` — unchanged (`"30m"`, `"2h"`)
- `ParseOrdinal(string raw)` — unchanged (`"last"`, `"last-1"`, integers)
- `ParseMonthlyDay(string raw)` — unchanged
- `ParseDateBound(string raw)` — unchanged
- `ParseDateOnly(string raw)` — unchanged
- `ParseDateTime(string raw)` — unchanged
- `ParsePositiveInt(string raw)` — unchanged (used by `everyWeeks`)
- `ParseSingleMonth(string raw)` — unchanged (called per array element)

**Contract (unchanged interface, same signature):**
```csharp
public interface IScheduleParser
{
    static abstract Result<ScheduleDefinition> Parse(string text);
}
```

---

## Phase 5 — JSON serializer

**Replaces the INI serializer. Delete:**
- `Parsing/ScheduleSerializer.cs`

**Rewrite:**
- `Parsing/ScheduleSerializer.cs` — public static class

`Serialize(ScheduleDefinition definition) → Result<string>` produces canonical JSONC:
a JSON array, indented 2 spaces, one object per rule, properties in definition order.
Uses `System.Text.Json.Utf8JsonWriter` directly (no custom reflection-based serializer —
the discriminated-union rule hierarchy does not map cleanly to `JsonSerializer.Serialize`
without a custom converter, and the writer approach is simpler and more explicit).

Comments are not emitted by the serializer — they are operator-authored and not round-tripped.

**New file:**
- `josyn-schedule.schema.json` — placed in the project root alongside the `.csproj`.
  Schema header includes: "This schema is normative for structure. Semantics are defined
  in ADR-026."

**Contract (unchanged interface):**
```csharp
public interface IScheduleSerializer
{
    static abstract Result<string> Serialize(ScheduleDefinition definition);
}
```

---

## Phase 6 — Semantic validator

**Status: done — field-name updates only.**

No structural changes. Update internal references:
- `active_from` / `active_until` string references → `activeFrom` / `activeUntil`
- `time` → `times` in error messages
- `WeekIntervalRule.Every` → `WeekIntervalRule.EveryWeeks` in the anchor-weekday check

**Contract and validation conditions: unchanged.**

---

## Phase 7 — Tests

**Rewrite parser and serializer tests. Validator and value-parser tests require minor updates only.**

**`ScheduleParserTests.cs` — full rewrite:**
- One happy-path JSONC input per rule type
- Missing required field → error
- Unknown `"type"` value → error
- Invalid leaf string values (bad time, bad date, bad duration) → error aggregation
- `//` comment lines → parsed without error
- `activeFrom` / `activeUntil` on a bounded rule

**`ScheduleSerializerTests.cs` — full rewrite:**
- Round-trip: parse JSONC → serialize → parse produces equal `ScheduleDefinition`
- Output is valid JSON (parseable by `JsonDocument.Parse` without comment handling)

**`ValueParserTests.cs` — trim:**
- Remove range-expression tests (`mon..fri`, `jan..jun`, comma-list parsing)
- Keep all leaf-value tests unchanged

**`ScheduleValidatorTests.cs` — minor updates:**
- Update inline JSONC strings to use new field names (`times`, `everyWeeks`, `activeFrom`, `activeUntil`)
- No logic changes

---

## Phase 8 — Documentation & docs-index

**Update:**
- `josyn-platform/repos/josyn-commons.md` — no change to the package row needed
- `josyn-platform/docs/docs-index.json` — no new entries; existing entries remain valid
- `josyn-platform/plans/ADR-026-schedule-definition-language-implementation.md` — this file
