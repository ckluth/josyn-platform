# ADR-026 — Schedule Definition Language

**Date:** 2026-06-19
**Updated:** 2026-06-19
**Status:** Accepted

---

## Context

ADR-024 established `TimeScheduler.exe` as the orchestrator responsible for time-based job
launches. It reads a schedule configuration and decides which jobs are due on each evaluation
tick. This ADR defines the **format and semantics of that configuration** — the schedule
definition language.

### Requirements

- Readable and editable by an operator without developer assistance.
- Lightweight for standard cases (daily interval, fixed time, monthly recurrence).
- Expressive enough for edge cases without resorting to a general-purpose language.
- Able to combine regular execution rules with explicit date-based exceptions.
- No external tooling required to author or read a schedule file.
- Structural validation available in standard editors without a dedicated validate command.

### Format choice: JSONC

JSONC (JSON with comments, parsed via `JsonCommentHandling.Skip` in `System.Text.Json`) was
chosen over INI, YAML, and TOML:

| Format | Assessment |
|--------|-----------|
| **JSONC** | `//` comments; standard JSON structure; editors validate against a companion JSON Schema file inline as the operator types; `System.Text.Json` handles all parsing — no custom preprocessor, continuation-folding logic, or hand-written serializer needed |
| INI-inspired | Compact `key = value` syntax with `#`/`;` comments; but requires a custom preprocessor, continuation-folding, block-splitting logic, and a hand-written serializer; no structural editor support |
| TOML | Has comments and richer types; but no standard schema validation mechanism; `[[array of tables]]` syntax adds cognitive load |
| YAML | Indentation-sensitive; whitespace errors produce silent misbehaviour; hostile to operators |

A companion `josyn-schedule.schema.json` file is distributed alongside the package. Editors
that support JSON Schema (VS Code, Rider) validate structure, required fields, and field
formats inline without running the parser.

`//` line comments are the only supported comment syntax. Block comments (`/* */`) are not
accepted — they add parsing complexity without meaningful benefit over multiple `//` lines.

All time values are compared against the **server's local machine date and time**. No
timezone configuration is supported or necessary — the server is always the reference clock.

---

## Decision

### Structure

A schedule file is a JSON array of **rule objects**. Each object has a `"type"` property
that discriminates the rule kind. The top-level array allows any mix of rule types in any
order. Operators use `//` comments to label rules; the parser ignores them.

### Rule types (`type`)

| Value | Purpose |
|-------|---------|
| `interval` | Repeat every N minutes or hours within a time window on specified days |
| `fixed` | Fire at one or more specific times on specified days |
| `nth_weekday` | Fire on the Nth (or last, or last-N) occurrence of a named weekday within a calendar period |
| `monthly_date` | Fire on a specific calendar day each month, with optional month filter |
| `week_interval` | Fire on specified days at specified times, repeating every N weeks |
| `once` | Fire exactly once at a specific date and time |
| `exclude` | Declare dates on which no rule in this file may fire |

---

## Vocabulary reference

This section is the authoritative reference for every keyword and value the language
accepts. No other values are valid.

### Day abbreviations (`days`, `weekday`)

| Keyword | Day |
|---------|-----|
| `mon` | Monday |
| `tue` | Tuesday |
| `wed` | Wednesday |
| `thu` | Thursday |
| `fri` | Friday |
| `sat` | Saturday |
| `sun` | Sunday |

### Day field syntax

The `days` field accepts either a named shorthand string or a JSON array of abbreviations:

| Form | Example | Meaning |
|------|---------|---------|
| Named shorthand | `"weekdays"` | Monday through Friday |
| Named shorthand | `"weekend"` | Saturday and Sunday |
| Named shorthand | `"daily"` | All seven days |
| Array | `["mon", "wed", "fri"]` | Explicit set |

Range syntax (`mon..fri`) is not accepted. Use a named shorthand or an explicit array.

### Month abbreviations (`months`)

| Keyword | Month |
|---------|-------|
| `jan` | January |
| `feb` | February |
| `mar` | March |
| `apr` | April |
| `may` | May |
| `jun` | June |
| `jul` | July |
| `aug` | August |
| `sep` | September |
| `oct` | October |
| `nov` | November |
| `dec` | December |

The `months` field is a JSON array of abbreviations (e.g. `["jan", "jul"]`). When omitted,
the rule applies to all months.

### Period values (`period`)

| Keyword | Meaning |
|---------|---------|
| `"month"` | Calendar month |
| `"quarter"` | Calendar quarter — Q1 = January, Q2 = April, Q3 = July, Q4 = October |
| `"year"` | Calendar year |

"Nth weekday of a quarter" means the Nth occurrence in the **first month** of that quarter.

### Ordinal values (`nth`)

| Value | Meaning |
|-------|---------|
| `1` … `5` | First through fifth occurrence within the period — **integer** |
| `"last"` | Last occurrence of that weekday within the period — **string** |
| `"last-1"` | Second-to-last occurrence — **string** |
| `"last-2"` | Third-to-last occurrence — **string** |

### Duration values (`every`)

Used only by the `interval` rule type.

| Form | Example | Meaning |
|------|---------|---------|
| String with `m` suffix | `"30m"` | 30 minutes |
| String with `h` suffix | `"2h"` | 2 hours |

Compound forms (e.g. `"1h30m"`) are not accepted. Use the smallest unit: `"90m"`, not `"1h30m"`.

### Key names

| Key | Used by | Type | Description |
|-----|---------|------|-------------|
| `type` | all | string | Rule type — one of the values in the type table above |
| `days` | `interval`, `fixed`, `week_interval` | string or array | Day shorthand or array of abbreviations |
| `start` | `interval` | string | Window open time — `"HH:mm"`, 24-hour |
| `end` | `interval` | string | Window close time — `"HH:mm"`, 24-hour |
| `every` | `interval` | string | Repetition interval — integer + duration suffix (`"30m"`, `"2h"`) |
| `everyWeeks` | `week_interval` | integer | Repetition interval in whole weeks |
| `anchor` | `week_interval` | string | ISO date (`"YYYY-MM-DD"`) that establishes the week-parity phase |
| `times` | `fixed`, `nth_weekday`, `week_interval` | array of strings | One or more `"HH:mm"` values |
| `weekday` | `nth_weekday` | string | Single day abbreviation |
| `nth` | `nth_weekday` | integer or string | Ordinal value (`1`–`5`, `"last"`, or `"last-N"`) |
| `period` | `nth_weekday` | string | Period value |
| `day` | `monthly_date` | integer or string | Calendar day — integer `1`–`31`, `"last"`, or `"last_business"` |
| `months` | `monthly_date` | array of strings | Month abbreviations; default when omitted is all months |
| `datetime` | `once` | string | Exact fire moment — `"YYYY-MM-DD HH:mm"` |
| `dates` | `exclude` | array | ISO date strings and/or range objects — see `exclude` type reference |
| `activeFrom` | any rule except `exclude` | string | Inclusive window start — `"YYYY-MM-DD"` (one-time) or `"MM-DD"` (annual) |
| `activeUntil` | any rule except `exclude` | string | Inclusive window end — `"YYYY-MM-DD"` (one-time) or `"MM-DD"` (annual) |

---

## Type reference

### `interval`

Repeat every fixed duration within a bounded time window on specified days.

The first fire is at `start`. Subsequent fires are at `start + N × every`. A computed slot
that falls after `end` is dropped silently — the window is never overrun. `end` itself is
a permitted fire time if it aligns exactly.

```jsonc
{
  "type": "interval",
  "days": "weekdays",
  "start": "08:00",
  "end": "17:00",
  "every": "30m"
}
```

### `fixed`

Fire at one or more explicit times on specified days. Each time value is an independent fire
trigger. Order within the array is irrelevant.

```jsonc
{
  "type": "fixed",
  "days": ["mon", "wed", "fri"],
  "times": ["06:00", "06:30"]
}
```

### `nth_weekday`

Fire on the Nth — or last, or last-N — occurrence of a named weekday within a calendar period.

**Exclusion collision:** if the computed date falls on an excluded date, that occurrence is
skipped entirely. No near-miss adjustment is performed. Silent date-shifting can produce
wrong business semantics; operators who need a fallback define an `once` rule explicitly.

```jsonc
// 2nd Tuesday of each month
{
  "type": "nth_weekday",
  "weekday": "tue",
  "nth": 2,
  "period": "month",
  "times": ["09:00"]
}

// 1st Monday of each quarter
{
  "type": "nth_weekday",
  "weekday": "mon",
  "nth": 1,
  "period": "quarter",
  "times": ["12:00"]
}

// Last Friday of each month
{
  "type": "nth_weekday",
  "weekday": "fri",
  "nth": "last",
  "period": "month",
  "times": ["16:00"]
}

// Second-to-last Friday of each month
{
  "type": "nth_weekday",
  "weekday": "fri",
  "nth": "last-1",
  "period": "month",
  "times": ["09:00"]
}
```

### `monthly_date`

Fire on a specific calendar day each month. The `months` key restricts which months are
active; when omitted, the rule fires every month.

`"day": "last"` fires on the last calendar day of the applicable months.
`"day": "last_business"` scans backward from the last calendar day, skipping weekends and
any date covered by an `exclude` rule in this file, and fires on the first valid day found.
If no valid day exists in the month (e.g. the entire final week is excluded), that month is
skipped silently.

If `day` is a number that exceeds the length of a given month (e.g. `31` in February),
the rule fires on the last calendar day of that month.

```jsonc
// 15th of every month
{
  "type": "monthly_date",
  "day": 15,
  "times": ["08:00"]
}

// Last calendar day of each month
{
  "type": "monthly_date",
  "day": "last",
  "times": ["17:30"]
}

// Last business day of each month
{
  "type": "monthly_date",
  "day": "last_business",
  "times": ["17:00"]
}

// Semi-annual — 1st of January and July
{
  "type": "monthly_date",
  "day": 1,
  "times": ["09:00"],
  "months": ["jan", "jul"]
}

// Bi-monthly — 1st of every other month
{
  "type": "monthly_date",
  "day": 1,
  "times": ["09:00"],
  "months": ["jan", "mar", "may", "jul", "sep", "nov"]
}
```

### `week_interval`

Fire on specified days at specified times, repeating every N weeks. The `anchor` key is a
past ISO date that was a valid fire date (i.e. a day matching the `days` field); it
establishes the phase. The scheduler fires on weeks where
`floor((today − anchor) / 7) mod everyWeeks == 0` and the weekday matches.

```jsonc
// Payroll run every other Friday
{
  "type": "week_interval",
  "days": ["fri"],
  "times": ["08:00"],
  "everyWeeks": 2,
  "anchor": "2026-01-02"
}

// Every 3 weeks on Monday and Thursday
{
  "type": "week_interval",
  "days": ["mon", "thu"],
  "times": ["10:00"],
  "everyWeeks": 3,
  "anchor": "2026-01-05"
}
```

### `activeFrom` / `activeUntil` (optional modifiers)

Any rule except `exclude` may carry these keys to restrict the date range during which
the rule is active. Outside the declared window the rule is ignored entirely — it is not
evaluated and does not contribute to launches.

Two forms are accepted for each key:

| Form | Example | Meaning |
|------|---------|---------|
| Full ISO date | `"2026-04-01"` | One-time boundary |
| Month-day | `"04-01"` | Recurring annual boundary (same date every calendar year) |

Both forms may be mixed on the same rule. An `"MM-DD"` `activeUntil` that is numerically
earlier than `activeFrom` correctly wraps across the year boundary
(e.g. `"activeFrom": "11-01", "activeUntil": "02-28"` means November through February).

```jsonc
// Every 30 min during business hours, but only in summer (annual, recurring)
{
  "type": "interval",
  "days": "weekdays",
  "start": "08:00",
  "end": "17:00",
  "every": "30m",
  "activeFrom": "04-01",
  "activeUntil": "09-30"
}

// One-time seasonal window
{
  "type": "fixed",
  "days": "daily",
  "times": ["06:00"],
  "activeFrom": "2026-11-01",
  "activeUntil": "2027-02-28"
}
```

### `once`

Fire exactly once at a specified date and time.

```jsonc
{
  "type": "once",
  "datetime": "2026-12-26 10:00"
}
```

### `exclude`

Declares dates on which **no rule in this file may fire**. Exclusions always win — no other
type can override them.

The `dates` array accepts ISO date strings (`"YYYY-MM-DD"`) and range objects
(`{ "from": "YYYY-MM-DD", "to": "YYYY-MM-DD" }`), in any combination. Multiple `exclude`
rules are allowed; their date sets are merged.

```jsonc
{
  "type": "exclude",
  "dates": [
    "2026-12-24",
    "2026-12-25",
    { "from": "2026-12-27", "to": "2026-12-31" }
  ]
}
```

---

## Composition semantics

Multiple rule objects in one file are **OR'd**: if any rule is satisfied at the current
evaluation moment, the job is triggered. Exclusions are applied after rule evaluation and
always win.

**Evaluation order does not matter** — two rules firing at the same moment do not produce
two launches. `TimeScheduler` emits one launch request per evaluation cycle per job,
regardless of how many rules are simultaneously satisfied.

---

## Complete example

```jsonc
[
  // Every 30 min during business hours, Mon–Fri
  {
    "type": "interval",
    "days": "weekdays",
    "start": "08:00",
    "end": "17:00",
    "every": "30m"
  },

  // Same interval but only during summer months (annual window)
  {
    "type": "interval",
    "days": "weekdays",
    "start": "06:00",
    "end": "08:00",
    "every": "30m",
    "activeFrom": "04-01",
    "activeUntil": "09-30"
  },

  // Fixed times on selected days
  {
    "type": "fixed",
    "days": ["mon", "wed", "fri"],
    "times": ["06:00", "06:30"]
  },

  // 2nd Tuesday of each month
  {
    "type": "nth_weekday",
    "weekday": "tue",
    "nth": 2,
    "period": "month",
    "times": ["09:00"]
  },

  // 1st Monday of each quarter
  {
    "type": "nth_weekday",
    "weekday": "mon",
    "nth": 1,
    "period": "quarter",
    "times": ["12:00"]
  },

  // Last Friday of each month
  {
    "type": "nth_weekday",
    "weekday": "fri",
    "nth": "last",
    "period": "month",
    "times": ["16:00"]
  },

  // Second-to-last Friday of each month
  {
    "type": "nth_weekday",
    "weekday": "fri",
    "nth": "last-1",
    "period": "month",
    "times": ["09:00"]
  },

  // 15th of every month
  {
    "type": "monthly_date",
    "day": 15,
    "times": ["08:00"]
  },

  // Last business day of each month
  {
    "type": "monthly_date",
    "day": "last_business",
    "times": ["17:00"]
  },

  // Semi-annual — 1st of January and July
  {
    "type": "monthly_date",
    "day": 1,
    "times": ["09:00"],
    "months": ["jan", "jul"]
  },

  // Payroll run every other Friday
  {
    "type": "week_interval",
    "days": ["fri"],
    "times": ["08:00"],
    "everyWeeks": 2,
    "anchor": "2026-01-02"
  },

  // One-off
  {
    "type": "once",
    "datetime": "2026-12-26 10:00"
  },

  // Exclusions
  {
    "type": "exclude",
    "dates": [
      "2026-12-24",
      "2026-12-25",
      { "from": "2026-12-27", "to": "2026-12-31" }
    ]
  }
]
```

---

## Alternatives considered

### Cron-only

A schedule file that is simply a list of cron expressions would be minimal and familiar to
system administrators. Rejected: cron syntax is opaque for non-developers, cannot express
natural-language patterns like "2nd Tuesday of the month" without complex DOM arithmetic,
and requires operators to know a format designed for Unix administrators — not the target
audience of this platform.

### INI-inspired

An INI-style `key = value` format was the original design (implemented and then replaced).
It produced a compact, comment-friendly file but required a custom preprocessor
(comment stripping, multi-line continuation folding), a hand-written block-splitting parser,
and a full serializer — all without any editor validation support. JSON Schema renders the
validation benefit reachable without that infrastructure cost.

### TOML

Has comments and richer types than INI. Rejected: no standard schema validation mechanism;
`[[array of tables]]` syntax for the list-of-rules structure adds cognitive load for operators.

### YAML

Indentation-sensitive; whitespace errors produce silent misbehaviour. Rejected without
further consideration.

### `..` range syntax in `days` / `months` arrays

Allowing `"mon..fri"` as a single array element was considered. Rejected: it requires the
same range-expansion parser as the INI design without adding expressive power — `"weekdays"`
covers the most common case, and an explicit array covers all others. Complexity not justified.

### Compound duration values

`"1h30m"` notation was considered and rejected. It requires a two-token parser. `"90m"` is
unambiguous and equally readable.

### Single `every` key for both `interval` and `week_interval`

The original INI design reused `every` on both rule types — a plain integer for weeks, a
suffixed string for sub-day intervals. In JSON, `everyWeeks` on `week_interval` makes the
unit self-documenting at the key level and allows the schema to apply different type
constraints (`integer` vs. `string` with pattern) without ambiguity.

### Block comments (`/* */`)

Not accepted — they add parsing complexity (`JsonCommentHandling` in `System.Text.Json` only
supports line comments) without meaningful benefit over multiple `//` lines.

---

## Consequences

- `TimeScheduler.exe` consumes this format. The parser reads a JSONC array via
  `System.Text.Json` with `JsonCommentHandling.Skip`, then dispatches on the `"type"`
  discriminator. The remaining custom parsing surface covers only leaf string values:
  `"HH:mm"` times, `"YYYY-MM-DD"` and `"MM-DD"` date bounds, `"YYYY-MM-DD HH:mm"`
  datetimes, `"Nm"`/`"Nh"` durations, ordinal strings (`"last"`, `"last-N"`),
  day abbreviations, month abbreviations, and `"last_business"`. All structural concerns
  (required fields, array shapes, type discrimination) are handled by the JSON layer.
- A `josyn-schedule.schema.json` file is distributed with the package. Operators point their
  editor at this schema to get inline validation.
- The file naming convention and storage location (per-job file vs. platform registry) are
  **not decided here**. That is a `TimeScheduler` implementation concern and will be
  addressed when the orchestrator is built.

---

## Open questions

1. **`once` state storage** — where is the "consumed" flag persisted? In the schedule file
   itself (a `"consumed": true` property written back), in the session store, or in a sidecar
   file? Not decided here; deferred to `TimeScheduler` implementation.

2. **Validation tooling** — should `josyn-toolbox` gain a `validate-schedule` command that
   parses and reports errors in a schedule file without running it? Anticipated as useful;
   not a prerequisite. The JSON Schema already covers structural validation in-editor.

