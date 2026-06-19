# ADR-026 — Schedule Definition Language

**Date:** 2026-06-19
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

### Format choice: INI-inspired

INI was chosen over JSON, YAML, and TOML:

| Format | Assessment |
|--------|-----------|
| **INI-inspired** | Operator-familiar `key = value` syntax; no quoting rules; comments with `#`/`;`; blank lines delimit rule blocks; already parsed by `BootstrapConfig` in a compatible form |
| TOML | Richer types, but introduces array and inline-table syntax that adds cognitive load for non-developer operators |
| YAML | Indentation-sensitive; whitespace errors produce silent misbehaviour; hostile to operators |
| JSON | No comments; verbose; not human-authoring-friendly |

`#` and `;` both serve as comment characters. Multi-value fields wrap with leading whitespace
(standard INI continuation).

All time values are compared against the **server's local machine date and time**. No
timezone configuration is supported or necessary — the server is always the reference clock.

---

## Decision

### Rule blocks

A schedule file is a sequence of **rule blocks**, separated by blank lines. A rule block is
a set of `key = value` lines. There are no named section headers — INI-style `[name]` syntax
is not used. The `type` key is the only structural discriminator. Operators use comments to
label blocks for their own readability; the parser ignores them.

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

This section is the authoritative reference for every keyword and abbreviation the language
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

### Day keywords (aliases)

| Keyword | Expands to |
|---------|-----------|
| `weekdays` | `mon..fri` |
| `weekend` | `sat, sun` |
| `daily` | `mon..sun` |

### Day expression syntax

| Form | Example | Meaning |
|------|---------|---------|
| Single abbreviation | `wed` | One specific day |
| Inclusive range | `mon..fri` | All days from Monday through Friday |
| Comma list | `mon, wed, fri` | Explicit set |
| Mix | `mon..wed, fri` | Range combined with individual day |

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

The `months` key uses the same expression syntax as `days`: single abbreviation, inclusive range
(`jan..jun`), comma list, or a mix. Default when omitted is all months.

### Period values (`period`)

| Keyword | Meaning |
|---------|---------|
| `month` | Calendar month |
| `quarter` | Calendar quarter — Q1 = January, Q2 = April, Q3 = July, Q4 = October |
| `year` | Calendar year |

"Nth weekday of a quarter" means the Nth occurrence in the **first month** of that quarter.

### Ordinal values (`nth`)

| Value | Meaning |
|-------|---------|
| `1` … `5` | First through fifth occurrence within the period |
| `last` | Last occurrence of that weekday within the period (not the last calendar day) |
| `last-1` | Second-to-last occurrence |
| `last-2` | Third-to-last occurrence |

### Duration suffixes (`every`)

| Suffix | Unit |
|--------|------|
| `m` | minutes |
| `h` | hours |

Compound forms (e.g. `1h30m`) are not accepted. Use the smallest unit: `90m`, not `1h30m`.

### Key names

| Key | Used by | Description |
|-----|---------|-------------|
| `type` | all | Rule type — one of the values in the type table above |
| `days` | `interval`, `fixed`, `week_interval` | Day expression |
| `start` | `interval` | Window open time — `HH:mm`, 24-hour |
| `end` | `interval` | Window close time — `HH:mm`, 24-hour |
| `every` | `interval` | Repetition interval — integer + duration suffix |
| `every` | `week_interval` | Repetition interval — plain integer (whole weeks, no suffix) |
| `anchor` | `week_interval` | ISO date (`YYYY-MM-DD`) that establishes the week-parity phase |
| `time` | `fixed`, `nth_weekday`, `week_interval` | One or more `HH:mm` values, comma-separated |
| `weekday` | `nth_weekday` | Single day abbreviation |
| `nth` | `nth_weekday` | Ordinal value (`1`–`5`, `last`, or `last-N`) |
| `period` | `nth_weekday` | Period value |
| `day` | `monthly_date` | Calendar day — integer `1`–`31`, `last`, or `last_business` |
| `months` | `monthly_date` | Month expression — same syntax as `days`; default is all months |
| `datetime` | `once` | Exact fire moment — `YYYY-MM-DD HH:mm` |
| `dates` | `exclude` | Comma-separated ISO dates and/or date ranges |
| `active_from` | any rule except `exclude` | Inclusive window start — `YYYY-MM-DD` (one-time) or `MM-DD` (annual) |
| `active_until` | any rule except `exclude` | Inclusive window end — `YYYY-MM-DD` (one-time) or `MM-DD` (annual) |

---

## Type reference

### `interval`

Repeat every fixed duration within a bounded time window on specified days.

The first fire is at `start`. Subsequent fires are at `start + N × every`. A computed slot
that falls after `end` is dropped silently — the window is never overrun. `end` itself is
a permitted fire time if it aligns exactly.

```
type  = interval
days  = weekdays
start = 08:00
end   = 17:00
every = 30m
```

### `fixed`

Fire at one or more explicit times on specified days. Each time value is an independent fire
trigger. Order within the list is irrelevant.

```
type = fixed
days = mon, wed, fri
time = 06:00, 06:30
```

### `nth_weekday`

Fire on the Nth — or last, or last-N — occurrence of a named weekday within a calendar period.

**Exclusion collision:** if the computed date falls on an excluded date, that occurrence is
skipped entirely. No near-miss adjustment is performed. Silent date-shifting can produce
wrong business semantics; operators who need a fallback define an `once` block explicitly.

```
type    = nth_weekday
weekday = tue
nth     = 2
period  = month
time    = 09:00

type    = nth_weekday
weekday = mon
nth     = 1
period  = quarter
time    = 12:00

type    = nth_weekday
weekday = fri
nth     = last
period  = month
time    = 16:00

# Second-to-last Friday of each month
type    = nth_weekday
weekday = fri
nth     = last-1
period  = month
time    = 09:00
```

### `monthly_date`

Fire on a specific calendar day each month. The `months` key restricts which months are
active; when omitted, the rule fires every month.

`day = last` fires on the last calendar day of the applicable months. `day = last_business`
scans backward from the last calendar day, skipping weekends and any date covered by an
`exclude` block in this file, and fires on the first valid day found. If no valid day exists
in the month (e.g. the entire final week is excluded), that month is skipped silently.

If `day` is a number that exceeds the length of a given month (e.g. `day = 31` in February),
the rule fires on the last calendar day of that month.

```
# 15th of every month
type = monthly_date
day  = 15
time = 08:00

# Last calendar day of each month
type = monthly_date
day  = last
time = 17:30

# Last business day of each month
type = monthly_date
day  = last_business
time = 17:00

# Semi-annual — 1st of January and July
type   = monthly_date
day    = 1
time   = 09:00
months = jan, jul

# Bi-monthly — 1st of every other month
type   = monthly_date
day    = 1
time   = 09:00
months = jan, mar, may, jul, sep, nov
```

### `week_interval`

Fire on specified days at specified times, repeating every N weeks. The `anchor` key is a
past ISO date that was a valid fire date (i.e. a day matching the `days` expression); it
establishes the phase. The scheduler fires on weeks where
`floor((today − anchor) / 7) mod every == 0` and the weekday matches.

`every` is a plain integer (whole weeks). No duration suffix is used.

```
# Payroll run every other Friday
type   = week_interval
days   = fri
time   = 08:00
every  = 2
anchor = 2026-01-02

# Every 3 weeks on Monday and Thursday
type   = week_interval
days   = mon, thu
time   = 10:00
every  = 3
anchor = 2026-01-05
```

### `active_from` / `active_until` (optional modifiers)

Any rule block except `exclude` may carry these keys to restrict the date range during which
the rule is active. Outside the declared window the rule is ignored entirely — it is not
evaluated and does not contribute to launches.

Two forms are accepted for each key:

| Form | Example | Meaning |
|------|---------|---------|
| Full ISO date | `2026-04-01` | One-time boundary |
| Month-day | `04-01` | Recurring annual boundary (same date every calendar year) |

Both forms may be mixed on the same rule. An `MM-DD` `active_until` that is numerically
earlier than `active_from` correctly wraps across the year boundary
(e.g. `active_from = 11-01, active_until = 02-28` means November through February).

```
# Run every 30 min during business hours, but only in summer (annual, recurring)
type         = interval
days         = weekdays
start        = 08:00
end          = 17:00
every        = 30m
active_from  = 04-01
active_until = 09-30

# One-time seasonal window
type         = fixed
days         = daily
time         = 06:00
active_from  = 2026-11-01
active_until = 2027-02-28
```

### `once`

Fire exactly once at a specified date and time.

```
type     = once
datetime = 2026-12-26 10:00
```

### `exclude`

Declares dates on which **no rule in this file may fire**. Exclusions always win — no other
`type` can override them.

The `dates` key accepts ISO dates (`YYYY-MM-DD`), inclusive date ranges
(`YYYY-MM-DD..YYYY-MM-DD`), and comma-separated combinations of both. Multi-line
continuation is permitted. Multiple `exclude` blocks are allowed; their date sets are merged.

```
type  = exclude
dates = 2026-12-24, 2026-12-25,
        2026-12-27..2026-12-31
```

---

## Composition semantics

Multiple rule blocks in one file are **OR'd**: if any rule is satisfied at the current
evaluation moment, the job is triggered. Exclusions are applied after rule evaluation and
always win.

**Evaluation order does not matter** — two rules firing at the same moment do not produce
two launches. `TimeScheduler` emits one launch request per evaluation cycle per job,
regardless of how many blocks are simultaneously satisfied.

---

## Complete example

```
# Every 30 min during business hours, Mon–Fri
type  = interval
days  = weekdays
start = 08:00
end   = 17:00
every = 30m

# Same interval but only during summer months (annual window)
type         = interval
days         = weekdays
start        = 06:00
end          = 08:00
every        = 30m
active_from  = 04-01
active_until = 09-30

# Fixed times on selected days
type = fixed
days = mon, wed, fri
time = 06:00, 06:30

# 2nd Tuesday of each month
type    = nth_weekday
weekday = tue
nth     = 2
period  = month
time    = 09:00

# 1st Monday of each quarter
type    = nth_weekday
weekday = mon
nth     = 1
period  = quarter
time    = 12:00

# Last Friday of each month
type    = nth_weekday
weekday = fri
nth     = last
period  = month
time    = 16:00

# Second-to-last Friday of each month
type    = nth_weekday
weekday = fri
nth     = last-1
period  = month
time    = 09:00

# 15th of every month
type = monthly_date
day  = 15
time = 08:00

# Last business day of each month
type = monthly_date
day  = last_business
time = 17:00

# Semi-annual — 1st of January and July
type   = monthly_date
day    = 1
time   = 09:00
months = jan, jul

# Payroll run every other Friday
type   = week_interval
days   = fri
time   = 08:00
every  = 2
anchor = 2026-01-02

# One-off
type     = once
datetime = 2026-12-26 10:00

# Exclusions
type  = exclude
dates = 2026-12-24, 2026-12-25,
        2026-12-27..2026-12-31
```

---

## Alternatives considered

### Cron-only

A schedule file that is simply a list of cron expressions would be minimal and familiar to
system administrators. Rejected: cron syntax is opaque for non-developers, cannot express
natural-language patterns like "2nd Tuesday of the month" without complex DOM arithmetic,
and requires operators to know a format designed for Unix administrators — not the target
audience of this platform.

### Special reserved section names

An earlier version of this design used `[exclude]` and `[meta]` as magic reserved section
names treated differently by the parser. Rejected: the uniform approach (`type = exclude`
like any other block) is simpler to parse, simpler to document, and removes an invisible
special case. Multiple exclusion blocks are a natural consequence of this choice.

### Per-rule exclusions

Allowing exclusion dates on individual rules was considered and rejected. The complexity of
tracking per-rule exclusion logic, combined with the rarity of the scenario, does not justify
the added surface area. Jobs that require rule-specific exclusions use separate files.

### Compound duration values

`1h30m` notation was considered and rejected. It requires a two-token parser. `90m` is
unambiguous and equally readable.

### Week-interval via duration suffix on `interval`

A `w` suffix on `every` (e.g. `every = 2w`) inside the existing `interval` type was
considered. Rejected: `interval` is inherently day-scoped — it owns `start` and `end`
time-of-day window keys. Mixing day-level and week-level semantics inside one type obscures
both. A separate `week_interval` type keeps each rule's contract self-contained.

---

## Consequences

- `TimeScheduler.exe` consumes this format. The parser must handle: blank-line-delimited
  rule blocks, the seven rule types, the day expression syntax (abbreviations, ranges, lists,
  keywords), month expression syntax (same rules), duration suffixes, the `..` range operator
  in `dates` values, `once` consumed-state tracking, `last_business` backward-scan logic,
  week-parity arithmetic for `week_interval`, and `active_from`/`active_until` window
  filtering on any rule.
- The file naming convention and storage location (per-job file vs. platform registry) are
  **not decided here**. That is a `TimeScheduler` implementation concern and will be
  addressed when the orchestrator is built.

---

## Open questions

1. **`once` state storage** — where is the "consumed" flag persisted? In the schedule file
   itself (a `consumed = true` line written back), in the session store, or in a sidecar
   file? Not decided here; deferred to `TimeScheduler` implementation.

2. **Validation tooling** — should `josyn-toolbox` gain a `validate-schedule` command that
   parses and reports errors in a schedule file without running it? Anticipated as useful;
   not a prerequisite.

