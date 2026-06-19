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
- All keywords and vocabulary in German — the platform's operator audience.

### Format choice: INI

INI was chosen over JSON, YAML, and TOML:

| Format | Assessment |
|--------|-----------|
| **INI** | Operator-familiar; no quoting rules; already parsed by `BootstrapConfig`; sections serve as natural grouping constructs for rule keys |
| TOML | Richer types, but introduces array and inline-table syntax that adds cognitive load for non-developer operators |
| YAML | Indentation-sensitive; whitespace errors produce silent misbehaviour; hostile to operators |
| JSON | No comments; verbose; not human-authoring-friendly |

`#` and `;` both serve as comment characters (standard INI). Multi-value fields wrap with
leading whitespace (standard INI continuation).

All time values are compared against the **server's local machine date and time**. No
timezone configuration is supported or necessary — the server is always the reference clock.

---

## Decision

### Sections

A schedule file is a collection of sections. Section names are structurally required by the
INI format to delimit one rule's keys from the next — the parser treats them as opaque.
Operators may choose descriptive names for their own readability; the language assigns no
meaning to them.

Every section must contain a `typ` key. The `typ` value determines which other keys are
valid for that section. There is no distinction between "rule sections" and "special
sections": exclusions are declared using `typ = ausschluss`, exactly like any other section.

### Rule types (`typ`)

| Value | Purpose |
|-------|---------|
| `intervall` | Repeat every N minutes or hours within a time window on specified days |
| `fest` | Fire at one or more specific times on specified days |
| `nth_wochentag` | Fire on the Nth (or last) occurrence of a named weekday within a calendar period |
| `einmalig` | Fire exactly once at a specific date and time |
| `ausschluss` | Declare dates on which no rule in this file may fire |

---

## Vocabulary reference

This section is the authoritative reference for every keyword and abbreviation the language
accepts. No other values are valid.

### Day abbreviations (`tage`, `wochentag`)

| Keyword | Meaning |
|---------|---------|
| `mo` | Montag |
| `di` | Dienstag |
| `mi` | Mittwoch |
| `do` | Donnerstag |
| `fr` | Freitag |
| `sa` | Samstag |
| `so` | Sonntag |

### Day keywords (aliases)

| Keyword | Expands to |
|---------|-----------|
| `wochentage` | `mo..fr` |
| `wochenende` | `sa, so` |
| `täglich` | `mo..so` |

### Day expression syntax

| Form | Example | Meaning |
|------|---------|---------|
| Single abbreviation | `mi` | One specific day |
| Inclusive range | `mo..fr` | All days from Monday through Friday |
| Comma list | `mo, mi, fr` | Explicit set |
| Mix | `mo..mi, fr` | Range combined with individual day |

### Period values (`periode`)

| Keyword | Meaning |
|---------|---------|
| `monat` | Calendar month |
| `quartal` | Calendar quarter — Q1 = Januar, Q2 = April, Q3 = Juli, Q4 = Oktober |
| `jahr` | Calendar year |

"Nth weekday of a quarter" means the Nth occurrence in the **first month** of that quarter.

### Ordinal values (`nth`)

| Value | Meaning |
|-------|---------|
| `1` … `5` | First through fifth occurrence within the period |
| `letzte` | Last occurrence of that weekday within the period (not the last calendar day) |

### Duration suffixes (`alle`)

| Suffix | Unit |
|--------|------|
| `m` | Minuten |
| `h` | Stunden |

Compound forms (e.g. `1h30m`) are not accepted. Use the smallest unit: `90m`, not `1h30m`.

### Key names

| Key | Used by | Description |
|-----|---------|-------------|
| `typ` | all | Rule type — one of the values in the type table above |
| `tage` | `intervall`, `fest` | Day expression |
| `start` | `intervall` | Window open time — `HH:mm`, 24-hour |
| `ende` | `intervall` | Window close time — `HH:mm`, 24-hour |
| `alle` | `intervall` | Repetition interval — integer + duration suffix |
| `zeit` | `fest`, `nth_wochentag` | One or more `HH:mm` values, comma-separated |
| `wochentag` | `nth_wochentag` | Single day abbreviation |
| `nth` | `nth_wochentag` | Ordinal value (`1`–`5` or `letzte`) |
| `periode` | `nth_wochentag` | Period value |
| `datum` | `einmalig` | Exact fire moment — `YYYY-MM-DD HH:mm` |
| `daten` | `ausschluss` | Comma-separated ISO dates and/or date ranges |

---

## Type reference

### `intervall`

Repeat every fixed duration within a bounded time window on specified days.

The first fire is at `start`. Subsequent fires are at `start + N × alle`. A computed slot
that falls after `ende` is dropped silently — the window is never overrun. `ende` itself is
a permitted fire time if it aligns exactly.

```ini
[tagespoll]
typ   = intervall
tage  = wochentage
start = 08:00
ende  = 17:00
alle  = 30m
```

### `fest`

Fire at one or more explicit times on specified days. Each time value is an independent fire
trigger. Order within the list is irrelevant.

```ini
[morgenverarbeitung]
typ  = fest
tage = mo, mi, fr
zeit = 06:00, 06:30
```

### `nth_wochentag`

Fire on the Nth — or last — occurrence of a named weekday within a calendar period.

**Exclusion collision:** if the computed date falls on an excluded date, that occurrence is
skipped entirely. No near-miss adjustment is performed. Silent date-shifting can produce
wrong business semantics; operators who need a fallback define an `einmalig` section
explicitly.

```ini
[monatsabschluss]
typ       = nth_wochentag
wochentag = di
nth       = 2
periode   = monat
zeit      = 09:00

[quartalskick]
typ       = nth_wochentag
wochentag = mo
nth       = 1
periode   = quartal
zeit      = 12:00

[letzter-freitag]
typ       = nth_wochentag
wochentag = fr
nth       = letzte
periode   = monat
zeit      = 16:00
```

### `einmalig`

Fire exactly once at a specified date and time. `TimeScheduler` marks the section as
consumed after it fires. The entry remains in the file but is not re-evaluated on subsequent
runs. Restarting the service does not re-fire it.

```ini
[jahreswechsel-sonderverarbeitung]
typ   = einmalig
datum = 2026-12-26 10:00
```

### `ausschluss`

Declares dates on which **no rule in this file may fire**. Exclusions always win — no other
`typ` can override them.

The `daten` key accepts ISO dates (`YYYY-MM-DD`), inclusive date ranges
(`YYYY-MM-DD..YYYY-MM-DD`), and comma-separated combinations of both. Multi-line
continuation is permitted. Multiple `ausschluss` sections are allowed; their date sets are
merged.

```ini
[feiertage]
typ   = ausschluss
daten = 2026-12-24, 2026-12-25,
        2026-12-27..2026-12-31
```

---

## Composition semantics

Multiple sections in one file are **OR'd**: if any rule is satisfied at the current
evaluation moment, the job is triggered. Exclusions are applied after rule evaluation and
always win.

**Evaluation order does not matter** — two rules firing at the same moment do not produce
two launches. `TimeScheduler` emits one launch request per evaluation cycle per job,
regardless of how many sections are simultaneously satisfied.

---

## Complete example

```ini
# ── Alle 30 Minuten während der Arbeitszeit, Mo–Fr ───────────────────────────
[tagespoll]
typ   = intervall
tage  = wochentage
start = 08:00
ende  = 17:00
alle  = 30m

# ── Feste Zeiten an ausgewählten Tagen ───────────────────────────────────────
[morgenverarbeitung]
typ  = fest
tage = mo, mi, fr
zeit = 06:00, 06:30

# ── 2. Dienstag im Monat ─────────────────────────────────────────────────────
[monatsabschluss]
typ       = nth_wochentag
wochentag = di
nth       = 2
periode   = monat
zeit      = 09:00

# ── 1. Montag im Quartal ─────────────────────────────────────────────────────
[quartalskick]
typ       = nth_wochentag
wochentag = mo
nth       = 1
periode   = quartal
zeit      = 12:00

# ── Letzter Freitag im Monat ─────────────────────────────────────────────────
[letzter-freitag]
typ       = nth_wochentag
wochentag = fr
nth       = letzte
periode   = monat
zeit      = 16:00

# ── Einmalige Sonderausführung ───────────────────────────────────────────────
[jahreswechsel]
typ   = einmalig
datum = 2026-12-26 10:00

# ── Ausschlüsse ──────────────────────────────────────────────────────────────
[feiertage]
typ   = ausschluss
daten = 2026-12-24, 2026-12-25,
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
names treated differently by the parser. Rejected: the uniform approach (`typ = ausschluss`
like any other section) is simpler to parse, simpler to document, and removes an invisible
special case. Multiple exclusion sections are a natural consequence of this choice.

### Per-rule exclusions

Allowing exclusion dates on individual rules was considered and rejected. The complexity of
tracking per-rule exclusion logic, combined with the rarity of the scenario, does not justify
the added surface area. Jobs that require rule-specific exclusions use separate files.

### Compound duration values

`1h30m` notation was considered and rejected. It requires a two-token parser. `90m` is
unambiguous and equally readable.

---

## Consequences

- `TimeScheduler.exe` consumes this format. The parser must handle: INI section parsing,
  the four rule types, the day expression syntax (abbreviations, ranges, lists, keywords),
  duration suffixes, the `..` range operator in `daten` values, and `einmalig`
  consumed-state tracking.
- The file naming convention and storage location (per-job file vs. platform registry) are
  **not decided here**. That is a `TimeScheduler` implementation concern and will be
  addressed when the orchestrator is built.

---

## Open questions

1. **`einmalig` state storage** — where is the "consumed" flag persisted? In the schedule
   file itself (a `verarbeitet = true` line written back), in the session store, or in a
   sidecar file? Not decided here; deferred to `TimeScheduler` implementation.

2. **Validation tooling** — should `josyn-toolbox` gain a `validate-schedule` command that
   parses and reports errors in a schedule file without running it? Anticipated as useful;
   not a prerequisite.
