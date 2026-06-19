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
| `nth_weekday` | Fire on the Nth (or last) occurrence of a named weekday within a calendar period |
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
| `days` | `interval`, `fixed` | Day expression |
| `start` | `interval` | Window open time — `HH:mm`, 24-hour |
| `end` | `interval` | Window close time — `HH:mm`, 24-hour |
| `every` | `interval` | Repetition interval — integer + duration suffix |
| `time` | `fixed`, `nth_weekday` | One or more `HH:mm` values, comma-separated |
| `weekday` | `nth_weekday` | Single day abbreviation |
| `nth` | `nth_weekday` | Ordinal value (`1`–`5` or `last`) |
| `period` | `nth_weekday` | Period value |
| `datetime` | `once` | Exact fire moment — `YYYY-MM-DD HH:mm` |
| `dates` | `exclude` | Comma-separated ISO dates and/or date ranges |

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

Fire on the Nth — or last — occurrence of a named weekday within a calendar period.

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
```

### `once`

Fire exactly once at a specified date and time. `TimeScheduler` marks the rule as consumed
after it fires. The entry remains in the file but is not re-evaluated on subsequent runs.
Restarting the service does not re-fire it.

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

---

## Consequences

- `TimeScheduler.exe` consumes this format. The parser must handle: blank-line-delimited
  rule blocks, the four rule types, the day expression syntax (abbreviations, ranges, lists,
  keywords), duration suffixes, the `..` range operator in `dates` values, and `once`
  consumed-state tracking.
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

