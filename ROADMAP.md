# JOSYN Platform — Roadmap & Status

> The Ariadne thread: where we are, where we came from, where we are going.
> Keep this file honest and current. It is not a specification — it is an orientation.

---

## The thread

JOSYN is a platform for executing scheduled jobs as isolated, observable processes.
The goal: a clean, maintainable replacement for legacy job infrastructure — with a proper
scheduler, a typed argument/result protocol, a persistent session store, and a clear
separation between platform concerns and job implementation.
The platform is being built incrementally, with a working round-trip already in place.

---

## Milestones

| # | Name | Status |
|---|------|--------|
| M1 | Foundation & core concepts | ✅ Done |
| M2 | Working round-trip — first Contoso job on a deployed system | ✅ Done |
| M3 | Scheduling — listener, ticker, time-based triggers | 🔄 In Progress |
| M4 | Full job lifecycle — migration, workflow support | ⬜ Upcoming |
| M5 | Production-ready platform | ⬜ Upcoming |

---

## What's done

- ADR consolidation — all backend ADRs merged into `josyn-platform/decisions/` using `ADR-NNNb-NN` interleaved naming
- Solid repo structure across all JOSYN repositories
- Extensive documentation — architecture, coding standards, decisions, repo summaries
- Core technical architecture defined and documented
- Strong set of coding principles, enforced via sanity checks
- `josyn-foundation` — settled: Result pattern, serialization, IPC transport
- `josyn-jap` — JAP protocol contracts and logging
- `josyn-job-host` — job execution runtime with arguments and result as basic features
- `josyn-backend` — growing: session store, error handler, job registry, session starter, backend CLI (first working implementation)
- ADR-009 ALC adapter model superseded by ADR-020 out-of-process adapter model; ALC artifacts removed
- `JOSYN.Backend.IdentityAdapter.Contract` in place; `Contoso.IdentityAdapter.exe` stub in `josyn-contoso`
- `josyn-contoso` — demo job in place; Contoso IdentityAdapter stub EXE replaces old ALC-based adapter
- Working end-to-end round-trip: a Contoso job spawned, executed, and reported by a deployed system
- Database in place
- `JOSYN.Backend.Ticker` — periodic process launcher; invokes registered orchestrators at configured intervals (ADR-024)
- `JOSYN.Commons.Schedule` — schedule definition language parser and evaluator; `ScheduleParser`, `ScheduleEvaluator`; all six rule types (ADR-026)
- `JOSYN.Backend.JobScheduleStore` — `IJobScheduleStore`, `IJobScheduleEntryRecord`, `IFiredSlotStore`; `josyn.JobSchedules`, `josyn.JobScheduleEntries`, `josyn.FiredSlots` tables (ADR-027, ADR-029)
- `JOSYN.Backend.JobRegistry` extended with `ArgumentRecords`; `IArgumentRecord`, `IJobRegistry.GetArgument()`; `josyn.ArgumentRecords` table (ADR-028)
- `JOSYN.Backend.TimeScheduler` — full tolerance-window + fired-slot-log scheduling algorithm; at-most-once delivery (ADR-027, ADR-028, ADR-029)

---

## What's in progress

- Backend CLI — first implementation done, must evolve
- Solution architecture documentation — large sections are still placeholders
- Documentation index tooling (`docs-index-builder`, AI enrichment pass pending)

---

## What's next

- Complete M3: implement `JOSYN.Backend.Listener` (the remaining piece; then M3 is done)
- Complete the solution architecture documentation
- Run AI enrichment pass on the documentation index

---

## Parked / out of scope (for now)

- Workflow support (multi-step jobs)
- Migration path from old jobs to the new platform
- Many `josyn-job-host` features beyond arguments/result basics
- Schema versioning and migration strategy (Flyway, DbUp, or equivalent) —
  define when the first shared persistent environment is established

---

## Open documentation items

Things that exist in the codebase but are not yet properly documented.
Work these off before a public release.

- **`josyn-backend` undocumented components** — four packages exist with no entry in
  `repos/josyn-backend/overview.md`:
  - `JOSYN.Backend.CLI` (`josyn-backend-cli`)
  - `JOSYN.Backend.Listener` (`josyn-backend-listener`)
  - `JOSYN.Backend.Ticker` (`josyn-backend-ticker`)
  - `JOSYN.Backend.ConfigStore` (`josyn-backend-config-store`)
  Each needs a component description, dependency list, and sanity notes entry.

- **`josyn-contoso`** — exists and works; no `repos/josyn-contoso.md` yet.
