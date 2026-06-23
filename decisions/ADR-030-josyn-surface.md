> **Correction (ADR-032, 2026-06-23):** Several references in this ADR to "the Listener stays
> a thin orchestrator" (D-17, SQ-2, D-19, Relation to Other ADRs, Continuation Brief) are
> superseded. No standalone `JOSYN.Backend.Listener` EXE is built. `start-session` is a verb
> on the surface agent's REST API. Individual spots are annotated inline below. See ADR-032.

> **Correction (ADR-033, 2026-06-23):** This ADR's D-3/D-17 **straddle** ("external tooling" vs.
> "platform-resident agent" — the admitted *"josyn-surface as a product straddles the boundary"*)
> is **resolved**, not merely refined. "josyn-surface" is now **three** distinct concerns:
> **(1)** the per-machine agent is renamed **`JOSYN.Backend.Gateway`** — a *platform*, mandatory
> component (no longer "surface"); **(2)** the cross-machine layer D-16 left unnamed is named
> **JRP — JOSYN Remote Protocol**, with contracts in a new contracts-only repo **`josyn-jrp`**
> (`JOSYN.Jrp.Launch` + `JOSYN.Jrp.Surface`) — this **closes the one remaining open naming loop
> (SQ-4)**; **(3)** only the *optional* edge clients keep the name "surface" (`josyn-surface`).
> Affected spots are annotated inline below. See ADR-033.

# ADR-030 — josyn-surface: The Human Window into a Headless Platform

**Date:** 2026-06-20
**Status:** Accepted (2026-06-21)

> Accepted 2026-06-21 after maintainer review, together with its delivery-strategy follow-up
> ADR-031. This ADR captures a shared vision, a role and use-case catalogue, an agreed set of
> high-level directional decisions, and the open questions that remain. It is deliberately
> coarse: it fixes *direction*, not *design*. The one remaining open loop is the *name* of the
> cross-machine layer; `josyn-surface` itself is provisional — a working title until a better
> one is chosen.

---

## Context

JOSYN is a **headless, server-side platform**. It runs as a set of decoupled machines, each
in a dedicated environment (`prod` | `int` | `dev`). A central scheduler spawns isolated job
processes; a session store and error store persist what happened; a config store and on-disk
bootstrap INI hold configuration. None of this has a human face. The only way a person sees
the platform today is by reading a database table or running a script.

During the current PoC phase, that human face has accreted as a **scattered set of tools** —
each addressing a single aspect of reporting, configuring, or editing:

| Tool (current) | Concern | Nature |
|----------------|---------|--------|
| `get-session-report` | Sessions | Read → markdown snapshot |
| `get-error-report` | Errors | Read → markdown snapshot |
| `josyn-db-viewer` | All tables | Read → renders HTML, opens browser |
| `insert-config-value` | ConfigStore | Write (hardcoded key/value) |
| `register-demo-job` | JobRegistry | Write (hardcoded job) |
| `arg-gen` | Argument records | Dev scaffolding (codegen) |
| `deploy-maintainer` | Deployment | Ops script |
| `users/*` | OS technical users | Infra script |

These tools are valuable precisely because each one is a **real use case someone already
needed**. But collectively they are uncoordinated: no shared access layer, no awareness of
environment or machine, no unifying view, no access control, and a mix of read and write with
no consistent safety stance.

The platform now needs a **vision for a surface** — a coherent, human-facing window onto the
headless realm. Until a better name exists, we call it **`josyn-surface`**.

This ADR does not design that surface. It establishes *what it is for*, *who it serves*, and
the *directional decisions* that constrain any future design.

---

## The Vision

> **josyn-surface is the human window into a headless platform.**

It is the single coherent place where a person can **observe** what the platform is doing
across all machines and environments, **investigate** what went wrong, and — deliberately and
safely — **control** what happens next. It replaces the scattered PoC tools' *capabilities*
(not their code) with a unified, environment-aware, access-tiered surface.

Two horizons:

- **Provisional product — the Reporting Surface.** Read-only. A single pane of glass across
  prod / int / dev that answers "what ran, what's running, what failed, what's scheduled, is
  everything OK?" Safe by construction (no mutations), it delivers value before the full
  vision exists.

- **Final product — the Control Center.** The Reporting Surface plus *gated* control: register
  and edit jobs, manage argument records and schedules, manually trigger and re-trigger jobs,
  suspend and resume schedules — each mutation deliberate, confirmable, audited, and access
  controlled, with prod held to the strictest gate.

---

## Roles

Six distinct roles drive the requirements. They differ along two axes: **trust/write-power**
and **technical depth**.

| # | Role | Who | Primary need |
|---|------|-----|-------------|
| R1 | **Maintainer — Operator** | Keeps the platform running; ops-minded | Health, investigation, light control |
| R2 | **Maintainer — Developer** | Builds and evolves the platform itself | Full read + full write |
| R3 | **Job Developer** | Writes job EXEs against josyn-job-host | Self-service registration, test, debug |
| R4 | **Scheduler Admin** | Owns the business schedule, not the code | Schedule visibility and management |
| R5 | **Auditor / Compliance** | Outside the team; evidence-gathering | Read-only, exportable trail |
| R6 | **Business Stakeholder** | Non-technical; outcome-only | "Did it run?" green/amber/red |

---

## Use Cases (the requirements anchor)

### R1 — Maintainer, Operator
- Overview of all machines × environments — "am I green?"
- List recent / in-flight / failed sessions; drill into any one.
- Inspect an error in full (message, call stack, exception details).
- Trace a session end-to-end (trigger → execution → result/error).
- Check orchestrator liveness (Ticker last fired when?).
- Light control: re-trigger a failed job, suspend / resume a schedule.

### R2 — Maintainer, Developer *(everything R1, plus)*
- Manage job registrations (add / edit / delete).
- Manage argument records (add / edit / delete).
- Manage schedules and entries (definition, tolerance, timing).
- Reset a fired slot (un-fire for re-execution); view the fired-slot log.
- Bootstrap / compare environments; inspect bootstrap INI vs ConfigStore.
- Trigger a job with ad-hoc arguments; view raw, untruncated payloads.
- Delete stale test data.

### R3 — Job Developer
- Self-service: register my job, define my argument records, define my schedule (in DEV).
- Trigger my job manually with specific arguments; see its result.
- See my job's errors in full; browse my job's session history.
- Iterate on argument values and schedule without involving the maintainer.

### R4 — Scheduler Admin
- See everything scheduled — all jobs, entries, timing.
- See what is planned next (today / this week) and what ran or was missed.
- Add / edit / suspend / resume schedule entries from the business calendar.
- Force a catch-up run outside the schedule.

### R5 — Auditor / Compliance
- Export session and error history for a date range.
- For a session: arguments, result, technical user, machine, outcome.
- Trace a session GUID from trigger to completion/failure.
- Read-only, export-friendly (printable / shareable).

### R6 — Business Stakeholder
- "Did job X run successfully today / this week?" — outcome only.
- Green / amber / red per job; no technical detail; self-service.

---

## Requirements

### Functional
- **FR-1** Cross-environment, cross-machine health overview.
- **FR-2** List/filter/drill sessions (running, completed, failed).
- **FR-3** Inspect errors in full detail.
- **FR-4** Trace a single session end-to-end.
- **FR-5** Show what is scheduled, what ran, what was missed.
- **FR-6** Show job registry, argument records, config store.
- **FR-7** Expose orchestrator / Ticker liveness.
- **FR-8** Manage job registrations.
- **FR-9** Manage argument records.
- **FR-10** Manage schedules and entries (incl. suspend/resume).
- **FR-11** Manually trigger a job with chosen arguments.
- **FR-12** Re-trigger / restart a failed session.
- **FR-13** Manage config store values.
- **FR-14** Reset fired-slots.
- **FR-15** Export sessions / errors / schedules for a date range.
- **FR-16** Environment / machine selection as a first-class navigation axis.
- **FR-17** Simple outcome view for non-technical stakeholders.

### Non-functional
- **NFR-1 Tiered access.** Read-only / operator / admin / developer tiers must be expressible,
  even if the PoC does not enforce them.
- **NFR-2 Mutation safety.** Writes — especially on prod — must be deliberate, confirmable, audited.
- **NFR-3 Environment isolation.** A prod action must never accidentally hit another environment;
  environment must be unmistakable in every view and action.
- **NFR-4 Multi-machine reach.** The surface addresses *n* machines, never assumes one.
- **NFR-5 No new coupling into the core.** The surface is a consumer; no platform runtime
  component may come to depend on it.
- **NFR-6 Reuse platform primitives.** Build on existing stores/contracts and the Result pattern;
  do not reinvent data access.
- **NFR-7 Incremental adoption.** The provisional product must deliver value standalone.
- **NFR-8 Low operational footprint.** The surface must not become a heavy system needing its own ops.

### Constraints (from the platform's nature)
- **C-1** Platform is headless server-side; the surface is the *only* human-facing window.
- **C-2** "Restart a job" means **spawning a process on a target machine**, not a DB write.
  Control actions cross the machine boundary and need an execution channel, not just data access.
- **C-3** Platform is functional-first C# (static-first, immutable, Result, no DI containers,
  no thrown exceptions). The surface must respect these standards where it touches platform code.
- **C-4** Data lives in **SQL Server per environment**; config is split between **ConfigStore (DB)**
  and **on-disk bootstrap INI**. The surface must reconcile both.

---

## Directional Decisions

The following directions were agreed during requirements analysis. Each fixes a heading, not a
detailed design.

| # | Question | Decision |
|---|----------|----------|
| D-1 | Product shape | **Family of thin shells over a shared backbone**, not one monolithic app. |
| D-2 | Read vs. write ownership | Surface owns **both**, but **write is a separate, deliberate shell** — not blended into the reporting view. |
| D-3 | Platform component vs. external | **External tooling** — alongside the platform (like the toolbox), not part of the runtime. *(Refined by D-17: this applies to the shells + aggregator; the per-machine agent is a platform-resident component.)* *(ADR-033: the straddle is dissolved — the per-machine agent is **platform** (renamed `JOSYN.Backend.Gateway`); only the **clients** are external. Three concerns, no straddle.)* |
| D-4 | Topology | **Central aggregator** — one surface reaches all machines. |
| D-5 | Control channel | A **resident agent / API on each machine** executes control actions locally. |
| D-6 | Cross-environment view | **Single pane of glass** across prod / int / dev. |
| D-7 | Technology shell | **Web (Blazor) primary** for reach (R4–R6) **+ CLI** for power ops (R1–R2). |
| D-8 | Data access | **API-mediated** via a platform-provided layer — not direct SQL from the surface. |
| D-9 | Auth / RBAC | **Design for it now, implement later.** The minimum must not be designed away. |
| D-10 | Prod mutation gate | **Explicit per-action confirmation + audit log.** |
| D-11 | Toolbox boundary | **Toolbox stays separate; surface is net-new.** Toolbox scripts inform requirements but their code is not inherited. |
| D-12 | Site-builder reuse | **Shared rendering building block where it fits** — chiefly the export/audit path, not the live dashboard. |
| D-13 | Repo | **Its own repo: `josyn-surface`.** |
| D-14 | Live updates | **Poll / refresh now; design for push later.** |
| D-15 | Machine discovery | A **central registry of machines / environments** the aggregator reads. |
| D-16 | Cross-machine transport (resolves SQ-4) | A **separate cross-machine layer** — an **HTTPS / REST API** exposed by each per-machine agent. **Wholly distinct from JIP**: distinct name, distinct contract, distinct concerns. JIP remains same-machine IPC and is untouched. *(ADR-033: this cross-machine layer is named **JRP — JOSYN Remote Protocol** (vs JIP = Interprocess); contracts live in `josyn-jrp` as `JOSYN.Jrp.Launch` + `JOSYN.Jrp.Surface`. Closes SQ-4's naming loop.)* |
| D-17 | Per-machine agent identity & placement (resolves SQ-2) | The per-machine agent is a **distinct sibling EXE**, **not** the evolved Listener — the Listener stays a thin orchestrator. *(The Listener is retired — see ADR-032. The agent is the sole network launch path.)* The agent is a **platform component, resident in `josyn-backend`** (it reads platform stores and spawns platform sessions). This **refines D-3**: only the shells + aggregator are "external"; the agent sits on the platform side. *(ADR-033: the agent is renamed **`JOSYN.Backend.Gateway`** and is unambiguously **platform** — the D-3/D-17 straddle is dissolved into three concerns.)* |
| D-18 | Machine/environment registry (resolves SQ-3) | A **central config file** on the aggregator side (honest about the bootstrap chicken-and-egg; matches ADR-010's flat-INI-per-installation spirit), **evolving toward a dedicated registry store** if metadata richness grows. **Not** one environment's ConfigStore (would silently re-couple environments). |
| D-19 | Launch-control verb (resolves SQ-6) | The agent **acts as its own orchestrator** for trigger / re-trigger: it constructs a `SessionStartRequest` and calls **`SessionLauncher` directly** — the sanctioned reuse point (ADR-017B-01). One hop, no dependency on the Listener being deployed. *(The Listener is retired — ADR-032. The agent is the sole network launch path; this decision stands unchanged.)* "Orchestrator" is a role the agent plays for one verb, not its identity. |
| D-20 | Functional-first scope in the shell (resolves SQ-5) | The **web shell and REST host are an accepted idiomatic-web exception zone** (DI, stateful components, lifecycle); the **backbone and agent domain/access logic hold full functional-first discipline** (C-3). The non-negotiable is a **stated boundary**: presentation/hosting = idiomatic web; domain/access logic = functional-first C#. |

---

## Emergent Architecture (provisional)

The decisions compose into a three-tier shape:

```
        ┌─────────────────────────────────────────────┐
        │   josyn-surface  (own repo, EXTERNAL)        │
        │                                              │
        │   Web shell (Blazor)        CLI shell        │
        │   R1, R4, R5, R6            R1, R2            │
        │        └──────────┬───────────┘              │
        │              shared backbone                 │
        │     (typed access + cross-machine aggregate) │
        └───────────────────┬──────────────────────────┘
                            │  HTTPS / REST (read + control)   ── D-16
        ════════════════════╪═════════════════ platform boundary ══
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
   ┌─────────┐         ┌─────────┐         ┌─────────┐
   │ agent   │         │ agent   │         │ agent   │   ← PLATFORM-RESIDENT
   │ PROD m1 │         │ INT  m2 │         │ DEV  m3 │     (josyn-backend EXE)
   │ DB+INI  │         │ DB+INI  │         │ DB+INI  │
   └────┬────┘         └─────────┘         └─────────┘
        │  launch control reuses the orchestrator path
        ▼
   SessionLauncher  *(Listener retired — ADR-032; agent is the sole network launch path)*
```

**Three durable components:**

1. **The per-machine surface agent** — a **platform-resident** EXE in `josyn-backend` (D-17),
   the trusted endpoint on each machine. Exposes the cross-machine HTTPS/REST API (D-16): reads
   the local DB and bootstrap INI, and performs control. For session-launch control it **reuses
   the orchestrator path** rather than reinventing it. *(The Listener is retired — ADR-032; the
   agent is the sole network launch path.)*

2. **The backbone** — typed access plus cross-machine aggregation; the durable shared
   investment beneath all shells. **External** (in the `josyn-surface` repo).

3. **The shells** — Blazor web (broad reach) and CLI (power ops); thin and replaceable.
   **External** (in the `josyn-surface` repo).

**Two-phase delivery:**

- **Provisional — Reporting Surface:** read-only, poll/refresh, single pane, no mutations.
- **Final — Control Center:** gated mutations, audit log, RBAC enforced, push updates.

---

## Open Questions

These arose *from* the directional decisions and are deliberately left open. They must be
resolved before design, not silently during it.

### SQ-1 — Agent and API: one component or two?
The per-machine agent (D-5) and the API layer (D-8) look like a single component. Confirm they
are unified, or articulate why they should be separated.

### SQ-2 — Agent vs. Listener ✅ Closed (see D-17) — *further superseded by ADR-032*

**Decision (original):** the per-machine agent is a **distinct sibling EXE**, **not** the evolved Listener.
The Listener stays a thin orchestrator (receive start-job request → `SessionLauncher`). The agent
is a **platform-resident component in `josyn-backend`** — it reads the platform stores and spawns
platform sessions, so it belongs on the platform side, not in the external `josyn-surface` repo.

> **Superseded (ADR-032, 2026-06-23):** The Listener is not built at all. The agent is not
> "distinct from the Listener" — it is the sole network launch path. The rationale below about
> not collapsing concerns into the Listener is retained for context but the conclusion changes:
> `start-session` folds into the agent as one verb.

This **refines D-3** without reversing it: the **shells + central aggregator** are external
tooling (the `josyn-surface` repo); the **per-machine agent** is a platform component. josyn-surface
as a *product* straddles the boundary — external consumers talking to a platform-resident agent.

Rationale: folding the agent's broad surface (read across all stores + admin writes across
registry/schedule/config + launch control) into the Listener would overload a deliberately-thin
orchestrator — the exact "collapse two concerns into one executable" mistake ADR-024 warns against.
~~Keeping them separate finishes M3's Listener as scoped and isolates the surface agent's concerns.~~
*(The Listener is retired — M3 is complete without it; the agent is the sole network launch path.)*

For **launch control** (trigger / re-trigger), the agent does not reinvent session-launch mechanics;
it reuses the orchestrator path — see SQ-6 for the remaining sub-decision.

### SQ-3 — Machine/environment registry ✅ Closed (see D-18)
**Decision:** a **central config file** on the aggregator side, evolving toward a dedicated registry
store if needed. The set of machines is small and changes rarely (≈ 3 environments × few machines),
so a flat aggregator-side file is honest about the bootstrap reality — the aggregator cannot query a
machine to learn that machines exist — and matches ADR-010's "environment = installation, flat INI"
philosophy. Rejected: designating one environment's `ConfigStore` as authoritative (C in the
analysis), which would secretly re-couple environments the platform works to keep separate, and make
the aggregator depend on one machine being up.

### SQ-4 — Cross-machine transport ✅ Closed (see D-16)

> **Naming closed (ADR-033):** the cross-machine layer is named **JRP — JOSYN Remote Protocol**
> (the `R`emote counterpart to JIP's `I`nterprocess). Its contracts live in the contracts-only repo
> **`josyn-jrp`** (`JOSYN.Jrp.Launch`, `JOSYN.Jrp.Surface`), and it is hosted by
> **`JOSYN.Backend.Gateway`**. The placeholder below ("its own name") is now resolved.

**Decision:** the surface reaches each machine over a **separate cross-machine layer** — an
**HTTPS / REST API** exposed by the per-machine agent — that is **wholly distinct from JIP**.

The platform's existing IPC (**JIP**) is **machine-bound by design**: named pipes, same machine,
broker ↔ job ↔ adapters. It is **not** extended across the network, it does **not** lend its name
or its message envelope to anything crossing a machine boundary, and it remains untouched by this
ADR.

The two boundaries get two distinct protocols:

| Layer | Boundary | Transport | Name |
|-------|----------|-----------|------|
| Existing IPC | Same machine | Named pipes | **JIP** |
| New surface layer | Cross machine | HTTPS, RESTful | *its own name — deliberately **not** JIP-anything* |

The cross-machine API is designed **on its own terms** — resources, verbs, status codes, auth —
as a network protocol, not as "JIP over HTTP." Conceptual cleanliness at the machine boundary
takes precedence over any internal code reuse: whatever the agent shares with JIP internally is an
implementation detail that never appears in the contract or the naming.

**Rationale for HTTPS/REST over alternatives:**
- The web shell already speaks HTTP; using it for the agent edge keeps **one transport family**
  end-to-end (browser → aggregator → agents).
- TLS plus a token (or mTLS) is the natural place where **auth and audit first bite** (D-9, D-10).
- Per-environment firewall rules (aggregator host → agent port) are normal, deliberate, documented.
- A message broker (RabbitMQ/MSMQ) adds standing infrastructure — rejected against NFR-8.
- gRPC adds proto tooling for no proportionate gain at this scale — rejected.
- Named pipes over SMB is cross-machine in theory but fragile, Windows-auth-bound, and cannot serve
  a browser — rejected, and would also violate the JIP-stays-local principle.

**Naming:** the cross-machine layer needs its **own name** (a placeholder until chosen) so it never
accidentally inherits "JIP." This is an explicit open naming item, not a protocol question.

### SQ-1 — Agent and API: one component or two? ✅ Closed (see D-16, D-17)
Resolved as a by-product of D-16 and D-17: the per-machine agent **is** the API. The agent (D-17)
is the platform-resident EXE that **exposes** the cross-machine HTTPS/REST API (D-16). They are one
component viewed from two angles — the resident process and the network endpoint it hosts — not two
separate things.

### SQ-5 — Functional-first standards in a web app ✅ Closed (see D-20)
**Decision:** the web shell and REST host are an **accepted idiomatic-web exception zone** (DI,
stateful components, lifecycle are the grain of Blazor/ASP.NET — fighting them yields worse code,
not more principled code). The **backbone and the agent's domain/access logic hold full
functional-first discipline** (C-3). The non-negotiable is a **stated boundary**:
presentation/hosting = idiomatic web; domain/access logic = functional-first C#. Without that line,
the exception decays into "web patterns everywhere."

### SQ-6 — Launch-control verb: delegate to Listener or call SessionLauncher directly? ✅ Closed (see D-19)
**Decision:** the agent **acts as its own orchestrator** — it constructs a `SessionStartRequest` and
calls **`SessionLauncher` directly**. ADR-017B-01 built `SessionLauncher` as *the* reusable launch
primitive precisely so any orchestrator can call it without duplicating session mechanics. This is
one hop, self-contained, and independent of whether a Listener is deployed on the machine. The mild
cost — the agent "becomes an orchestrator" — is acceptable: orchestrator = "constructs
`SessionStartRequest`, calls `SessionLauncher`," a role the agent plays for one verb, not its
identity. *(The Listener is retired — ADR-032. The "delegate to Listener" option is moot; the
agent's direct `SessionLauncher` call is now the only network-launch path.)*

---

## Consequences

- A new repo `josyn-surface` is anticipated (D-13) once this proposal is accepted. It holds the
  **external** parts: the web shell, the CLI shell, and the central aggregator.
- The platform (`josyn-backend`) gains a **distinct, platform-resident per-machine surface agent**
  EXE (D-17), exposing a **new cross-machine HTTPS / REST API** (D-16). The Listener is **not**
  overloaded — it stays a thin orchestrator, and M3 can finish it as scoped.
  **JIP is untouched**: it remains same-machine IPC. The cross-machine layer is a distinct protocol
  with its own name and contract.
- A **machine/environment registry** becomes a first-class platform artefact (SQ-3).
- The scattered PoC tools are reframed as a **requirements catalogue**, not a codebase to absorb
  (D-11). The toolbox continues to exist for maintainer tooling.
- `site-builder` is positioned as a **shared rendering building block** for the export/audit path
  (D-12).
- Access control and audit become **designed-in concerns from day one**, even though enforcement
  is deferred (D-9, D-10).

---

## Status & Next Steps

This ADR is **Accepted** (2026-06-21, maintainer review). All six sub-questions raised
during analysis are resolved: **SQ-1** (agent *is* the API), **SQ-2** (distinct sibling,
platform-resident), **SQ-3** (aggregator-side config file → store), **SQ-4** (HTTPS/REST, distinct
from JIP), **SQ-5** (shell is an idiomatic-web exception zone; backbone holds the line), **SQ-6**
(agent calls `SessionLauncher` directly). The one remaining item is the **name** for the
cross-machine layer (a placeholder until chosen) — a naming task, not an open architectural
question. *(Closed by ADR-033: the layer is **JRP — JOSYN Remote Protocol**, hosted by
`JOSYN.Backend.Gateway`, contracts in `josyn-jrp`.)* Delivery method is settled by the follow-up **ADR-031**; implementation begins with its
scoped MVP-1.

---

## Relation to Other ADRs

- **ADR-024 (Ticker)** and the **TimeScheduler** chain (ADR-026/027/028/029) produce the
  scheduling data the surface visualises and manages.
- **ADR-025 (SessionBroker)** and the session/error stores produce the runtime data the surface
  reports on.
- The planned **`JOSYN.Backend.Listener`** (roadmap M3) ~~stays a **thin orchestrator** and is
  *not* the per-machine agent (D-17). The agent is a distinct, platform-resident sibling EXE.~~
  *(Retired — ADR-032. No standalone Listener is built. `start-session` is a verb on the agent's
  REST API.)*
- **ADR-010 (Environment separation)** underpins the environment-as-first-class-axis requirement
  (FR-16, NFR-3).

---

## Continuation Brief

> A cold-start orientation for any follow-up session. Read this block first to know exactly where
> the conversation stopped and what the next moves are — without reverse-engineering them from the body.

**State of play.** This proposal is *complete as a directional ADR*. Vision, six roles (R1–R6),
the full use-case catalogue, requirements (FR/NFR/constraints), and **twenty directional decisions
(D-1 … D-20)** are settled. **All six sub-questions (SQ-1 … SQ-6) are closed** with rationale. The
**single open loop** is the *name* of the cross-machine layer — a naming task, not an architectural
question.

**To resume the conversation.** Read this ADR plus the `AGENTS.md` knowledge map. The load-bearing
prerequisite ADRs — open them if a decision here needs grounding — are:
- **ADR-010** (environment = installation; flat INI per install) → underpins D-18.
- **ADR-017B-01** (`SessionLauncher`, `SessionStartRequest`, the orchestrator model) → underpins D-19.
- **ADR-024** (Ticker vs. orchestrators; the Listener's thin role) → underpins D-17.
This ADR deliberately does not restate those facts (ADR-005: no duplication, read the canonical source).

**The decided architecture in one breath.** External shells (Blazor web + CLI) and a central
aggregator — all in a new **`josyn-surface`** repo — reach, over **HTTPS/REST** (distinct from JIP),
a **platform-resident per-machine agent EXE** that lives in **`josyn-backend`**. The agent *is* the
API; it reads the local stores + bootstrap INI, performs admin writes, and for trigger/re-trigger
calls **`SessionLauncher` directly**. The aggregator learns its machines from an **aggregator-side
config file**. ~~The Listener stays a thin orchestrator and is *not* the agent.~~ *(Listener
retired — ADR-032. The agent is the sole network launch path.)*

**On acceptance, the work (not yet started — none of this exists):**
1. Create the **`josyn-surface`** repo (web shell, CLI shell, central aggregator).
2. Add the **platform-resident agent EXE** to `josyn-backend` (new Pattern-B sub-folder; ~~sibling to Listener~~ *(Listener retired — ADR-032)*). 
3. Write the follow-up ADRs this proposal spawns:
   - the **cross-machine REST API contract** — *and* coin its **name** (the open loop).
   - the **auth / RBAC model** (D-9, D-10) — designed-in, deferred enforcement.
   - the **agent EXE spec** (endpoints, store access, launch-control via `SessionLauncher`).
4. Update **`ROADMAP.md`**, **`docs/docs-index.json`**, and **`repos/josyn-backend/overview.md`**
   (the new agent EXE) — currently untouched, correct while this is a proposal.

**Promotion.** Accepted 2026-06-21 (with ADR-031). Implementation proceeds via ADR-031's scoped
MVP-1; the work list above is the broader backlog beyond that first increment.
