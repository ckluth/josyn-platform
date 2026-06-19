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

### Format choice: INI

INI was chosen over JSON, YAML, and TOML:

| Format | Assessment |
|--------|-----------|
| **INI** | Operator-familiar; no quoting rules; already parsed by `BootstrapConfig`; section names become rule identifiers naturally |
| TOML | Richer types, but introduces array and inline-table syntax that adds cognitive load for non-developer operators |
| YAML | Indentation-sensitive; whitespace errors produce silent misbehaviour; hostile to operators |
| JSON | No comments; verbose; not human-authoring-friendly |

`#` and `;` both serve as comment characters (standard INI). Multi-value fields wrap with
leading whitespace (standard INI continuation).

---

## Decision

### Rule sections

A schedule file is a collection of **named rule sections**. The section name is the logical
identifier for the rule — it appears in logs and diagnostics. Any name is valid as long as it
is unique within the file and does not collide with the two reserved names `exclude` and
`meta` (see below).

Every rule section must contain a `type` key. The `type` determines which other keys are
valid for that section.

### Rule types

| `type` | Purpose |
|--------|---------|
| `interval` | Repeat every N minutes or hours within a time window on specified days |
| `fixed` | Fire at one or more specific times on specified days |
| `nth_weekday` | Fire on the Nth (or last) occurrence of a named weekday within a calendar period |
| `once` | Fire exactly once at a specific date and time |
| `cron` | Raw five-field cron expression — escape hatch for cases the above cannot cover |

---

### Day expressions

All types that accept a `days` key share the same day expression syntax.

| Expression | Meaning |
|------------|---------|
| `mon` `tue` `wed` `thu` `fri` `sat` `sun` | Single named day (lowercase, three letters) |
| `mon..fri` | Inclusive range — the `..` range operator |
| `mon, wed, fri` | Explicit comma-separated list |
| `weekdays` | Keyword alias for `mon..fri` |
| `weekend` | Keyword alias for `sat, sun` |
| `daily` | Keyword alias for `mon..sun` |

Ranges and lists may be combined: `mon..wed, fri`.

---

### Type reference

#### `interval`

Repeat every fixed duration within a bounded time window on specified days.

| Key | Required | Description |
|-----|----------|-------------|
| `days` | yes | Day expression |
| `start` | yes | Window open time — `HH:mm`, 24-hour |
| `end` | yes | Window close time — `HH:mm`, 24-hour |
| `every` | yes | Repetition interval — integer followed by `m` (minutes) or `h` (hours) |

The first fire is at `start`. Subsequent fires are at `start + N * every`. A computed slot
that falls after `end` is dropped silently — the window is never overrun. `end` itself is
a permitted fire time if it aligns exactly.

```ini
[business-hours-polling]
type  = interval
days  = mon..fri
start = 08:00
end   = 17:00
every = 30m
```

#### `fixed`

Fire at one or more explicit times on specified days.

| Key | Required | Description |
|-----|----------|-------------|
| `days` | yes | Day expression |
| `time` | yes | One or more `HH:mm` values, comma-separated |

Each time value is an independent fire trigger. Order within the list is irrelevant.

```ini
[morning-batch]
type = fixed
days = mon, wed, fri
time = 06:00, 06:30
```

#### `nth_weekday`

Fire on the Nth — or last — occurrence of a named weekday within a calendar period.

| Key | Required | Description |
|-----|----------|-------------|
| `weekday` | yes | Single named day (`mon` … `sun`) |
| `nth` | yes | Ordinal position: `1`, `2`, `3`, `4`, `5`, or `last` |
| `period` | yes | Calendar period: `month`, `quarter`, or `year` |
| `time` | yes | One or more `HH:mm` values, comma-separated |

**`period = quarter`** aligns to calendar quarters: Q1 = January, Q2 = April, Q3 = July,
Q4 = October. "Nth weekday of the quarter" means the Nth occurrence in the first month of
that quarter.

**`nth = last`** resolves to the last occurrence of the named weekday within the period —
not the last calendar day of the period.

**Exclusion collision:** if the computed date falls on a globally excluded date, that
occurrence is skipped entirely. No near-miss adjustment is performed (no "find next valid
day" fallback). This is intentional: silent date-shifting can produce wrong business
semantics. Operators who need a fallback should define a `once` rule explicitly.

```ini
[monthly-review]
type    = nth_weekday
weekday = tue
nth     = 2
period  = month
time    = 09:00

[quarterly-kickoff]
type    = nth_weekday
weekday = mon
nth     = 1
period  = quarter
time    = 12:00

[end-of-month-report]
type    = nth_weekday
weekday = fri
nth     = last
period  = month
time    = 16:00
```

#### `once`

Fire exactly once at a specified date and time.

| Key | Required | Description |
|-----|----------|-------------|
| `datetime` | yes | `YYYY-MM-DD HH:mm` |

`TimeScheduler` marks a `once` rule as consumed after it fires. The entry remains in the
file but is not re-evaluated on subsequent runs. Restarting the service does not re-fire it.

```ini
[year-end-special]
type     = once
datetime = 2026-12-26 10:00
```

#### `cron`

Raw five-field cron expression. Provides an escape hatch for schedules that cannot be
expressed with the types above.

| Key | Required | Description |
|-----|----------|-------------|
| `expr` | yes | Standard five-field cron expression (`minute hour dom month dow`) |

The `cron` type does not accept a `days` key or any other modifier — the expression is
evaluated as-is. Exclusions still apply.

```ini
[legacy-compatible]
type = cron
expr = 0 6 1-7 * 1     # first Monday of every month at 06:00
```

---

### Special sections

#### `[exclude]`

Declares date exclusions that apply globally to all rules in the file. Exclusions always
win — no rule type can override them.

| Key | Accepts |
|-----|---------|
| `dates` | ISO dates (`YYYY-MM-DD`), inclusive date ranges (`YYYY-MM-DD..YYYY-MM-DD`), or a comma-separated mix of both |

Multi-line continuation is permitted:

```ini
[exclude]
dates = 2026-12-24, 2026-12-25,
        2026-12-27..2026-12-31
```

One `[exclude]` section per file. If a job needs different exclusion sets for different
rules, it should use separate schedule files.

#### `[meta]`

Optional file-level metadata.

| Key | Default | Description |
|-----|---------|-------------|
| `timezone` | Server local time | IANA timezone identifier (e.g. `Europe/Berlin`) |

```ini
[meta]
timezone = Europe/Berlin
```

`[meta]` is reserved for future extension. `TimeScheduler` reads `timezone` if present and
applies it when evaluating all time values in the file. If absent, server local time is used.

---

### Composition semantics

Multiple rules in one file are **OR'd**: if any rule fires at the current evaluation moment,
the job is triggered. Exclusions are applied after rule evaluation and always win.

**Evaluation order does not matter** — two rules firing at the same moment do not produce
two launches. `TimeScheduler` emits one launch request per evaluation cycle per job,
regardless of how many rules are satisfied simultaneously.

---

### Duration syntax (`every` key)

| Input | Meaning |
|-------|---------|
| `15m` | 15 minutes |
| `2h` | 2 hours |
| `90m` | 90 minutes |

Compound forms (e.g. `1h30m`) are not accepted. Use the smallest unit: `90m`, not `1h30m`.

---

### Complete example

```ini
# ── File-level metadata ───────────────────────────────────────────────────────
[meta]
timezone = Europe/Berlin

# ── Repeat every 30 min during business hours, Mon–Fri ───────────────────────
[business-hours]
type  = interval
days  = mon..fri
start = 08:00
end   = 17:00
every = 30m

# ── Fixed times on specific days ─────────────────────────────────────────────
[morning-batch]
type = fixed
days = mon, wed, fri
time = 06:00, 06:30

# ── 2nd Tuesday of each month ────────────────────────────────────────────────
[monthly-review]
type    = nth_weekday
weekday = tue
nth     = 2
period  = month
time    = 09:00

# ── 1st Monday of each quarter ───────────────────────────────────────────────
[quarterly-kickoff]
type    = nth_weekday
weekday = mon
nth     = 1
period  = quarter
time    = 12:00

# ── Last Friday of each month ────────────────────────────────────────────────
[end-of-month-report]
type    = nth_weekday
weekday = fri
nth     = last
period  = month
time    = 16:00

# ── One-off ───────────────────────────────────────────────────────────────────
[year-end-special]
type     = once
datetime = 2026-12-26 10:00

# ── Escape hatch for an unusual pattern ──────────────────────────────────────
[legacy-compatible]
type = cron
expr = 0 6 * * 1-5

# ── Global exclusions ────────────────────────────────────────────────────────
[exclude]
dates = 2026-12-24, 2026-12-25,
        2026-12-27..2026-12-31
```

---

## Alternatives considered

### Cron-only

A schedule file that is simply a list of cron expressions would be minimal and familiar to
system administrators. Rejected: cron syntax is opaque for non-developers and cannot express
natural-language patterns like "2nd Tuesday of the month" without complex DOM arithmetic.
The `cron` type is retained as an escape hatch only.

### XML / custom DSL

A custom grammar (e.g. `every 30 minutes on weekdays from 08:00 to 17:00`) is maximally
readable but requires a non-trivial parser and produces ambiguity. INI's structured key-value
form is unambiguous without a grammar.

### Per-rule exclusions

Allowing `exclude_dates` on individual rules was considered and rejected. The complexity of
tracking per-rule consumed state, combined with the rarity of the scenario, does not justify
the added surface area. Jobs that require rule-specific exclusions use separate files.

### Compound duration values

`1h30m` notation was considered and rejected. It requires a two-token parser. `90m` is
unambiguous and just as readable.

---

## Consequences

- `TimeScheduler.exe` consumes this format. The parser must handle: INI section parsing,
  the five rule types, the day expression syntax (ranges, lists, keywords), duration suffixes,
  the `..` range operator in exclusion dates, and the `once` consumed-state tracking.
- The file naming convention and storage location (per-job file vs. platform registry) are
  **not decided here**. That is a `TimeScheduler` implementation concern and will be addressed
  when the orchestrator is built.
- The `[meta] timezone` key is reserved but `TimeScheduler` may treat it as a no-op until
  multi-timezone support is explicitly needed.

---

## Open questions

1. **`once` state storage** — where is the "consumed" flag persisted? In the schedule file
   itself (a `consumed = true` line written back), in the session store, or in a sidecar file?
   Not decided here; deferred to `TimeScheduler` implementation.

2. **Validation tooling** — should `josyn-toolbox` gain a `validate-schedule` command that
   parses and reports errors in a schedule file without running it? Anticipated as useful;
   not a prerequisite.
