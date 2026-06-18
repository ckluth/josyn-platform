# ADR-025 — SessionBroker: Rename JAPServer and Extract to Dedicated Repo

**Date:** 2026-06-18
**Status:** Accepted
**Supersedes:** ADR-004 (partially — see below)

---

## Context

`JOSYN.Jap.JAPServer` is the per-session process that sits between `josyn-backend` and
`job.exe`. It is the most architecturally significant executable in the platform: it is
the **only** component that simultaneously touches both worlds — the maintainer's backend
world and the job developer's execution world.

### The two-worlds model

```
══════════════════════════════════════════════════════════════════
  WORLD 1 — The Job Developer's World
──────────────────────────────────────────────────────────────────
  job.exe
    └── Core.Run(args)    (JOSYN.JobHost)
          └── [JobEntryPoint]   plain C# record in/out
              (no knowledge of backend, sessions, or scheduler)
══════════════════════════════════════════════════════════════════
                     ▲  ▼  (GUID-named pipes, JAP protocol)
          ┌──────────────────────────────┐
          │   JOSYN.Jap.JAPServer        │  ← the boundary process
          └──────────────────────────────┘
                     ▲  ▼  (SessionStore, credentials, config)
══════════════════════════════════════════════════════════════════
  WORLD 2 — The Maintainer's Backend World
──────────────────────────────────────────────────────────────────
  josyn-backend
    ├── SessionStore    (SQL Server session persistence)
    ├── BootstrapConfig (INI-based runtime config)
    ├── SessionStarter  (GUID assignment, session record, spawn)
    └── JobRegistry     (job name → technical user mapping)
══════════════════════════════════════════════════════════════════
```

Every other platform component is fully inside one world. This process alone straddles both.

### The four problems with "JAPServer"

**1. Opacity.** "JAP" is an acronym abbreviating another acronym plus two words. A future
maintainer encountering `JAPServer.exe` in a process list, or `JOSYN.Jap.JAPServer` in a
namespace, has no semantic entry point without prior architectural context. The name gives
nothing away about what the process does or why it matters.

**2. "Server" misleads about the component's dominant characteristic.** A server is a
passive responder to requests. This process:
- Spawns adapter EXEs
- Spawns `job.exe`
- Reads credentials and session data from the store
- Negotiates acceptance/rejection with the job
- Manages adapter lifecycle for the session duration
- Writes results and errors back to the session store

That is an active per-session orchestrator. "Server" names one technical dimension — that it
serves the JAP protocol to the job — while hiding the dominant characteristic.

**3. False symmetry with `JAPClient`.** Because there is a `JAPServer`, there appears to be a
symmetric peer: `JAPClient` in `josyn-job-host`. The Server ↔ Client framing implies two sides
of a peer relationship. The asymmetry is profound: one is backend infrastructure with platform
authority over a session; the other is a consumer-facing SDK for job developers. The current
naming actively obscures the two-worlds model.

**4. Namespace tension.** `JOSYN.Jap.JAPServer` living in `josyn-backend` is a known
incoherence, accepted by ADR-004 Challenge 5 on the grounds that "the namespace reflects what
the component *is* — the JAP server." With the renaming, that argument dissolves.

### The location problem

ADR-004 moved `JAPServer` from `josyn-jap` into `josyn-backend` because it needs
`SessionStore` and `BootstrapConfig` — resources owned by the backend. Keeping it in
`josyn-jap` would have required a reverse dependency (lower layer depending on outer layer),
an architectural violation.

The solution was relocation. But `josyn-backend` is designed as the **scheduler and
orchestration layer**. Hosting the per-session boundary EXE there conflates two distinct
concerns: orchestration infrastructure (Ticker, SessionStarter, JobRegistry) and the
per-session execution bridge.

More critically: the location buries the most architecturally significant EXE in the platform
as a subfolder of a larger repo, with no independent identity, no independent version history,
and no structural symmetry with its counterpart on the other side of the boundary —
`josyn-job-host`.

---

## Decision

**1. Rename `JOSYN.Jap.JAPServer` to `JOSYN.SessionBroker`.**

The new name:
- "Session" — immediately scopes it: one per execution session, ephemeral
- "Broker" — mediates between two parties, trusted by both, manages the terms of exchange

**2. Extract `JOSYN.SessionBroker` into a dedicated repo: `josyn-session-broker`.**

The repo consumes `josyn-backend`'s library packages via NuGet — the same pattern already
practiced across the entire platform. No new mechanism is required.

---

## Naming alternatives considered

| Candidate | Rejected because |
|-----------|-----------------|
| `SessionConductor` | "Conductor" is expressive but uncommon in software vocabulary; "Broker" is a well-established architectural pattern word |
| `ExecutionGateway` | "Gateway" carries strong network/API infrastructure connotations; invites confusion with API gateways |
| `ExecutionHost` | Collides conceptually with `josyn-job-host` — "host" is already the job-developer-side word |
| `SessionAgent` | "Agent" is already in use in the legacy backend (`TriggerAgent`); risks collision |
| `SessionConductor` | Expressive, but "conducting" implies event-driven flow; this process manages a session lifecycle, not an event stream |
| Keep `JAPServer` | Retains all four problems listed in context |

`SessionBroker` is the strongest candidate: it is self-explanatory, architecturally precise
(brokering between two worlds is exactly the role), and carries no dangerous vocabulary
collisions.

---

## Namespace decision

`JOSYN.Jap.JAPServer` carried the `Jap` layer segment because ADR-004 Challenge 5 argued that
the namespace should reflect what the component *is* — the JAP server. With the rename, that
argument no longer applies: `JOSYN.Jap.SessionBroker` would say "a session broker in the Jap
layer," which is neither precise nor useful.

Three options:

| Option | Assessment |
|--------|-----------|
| `JOSYN.Jap.SessionBroker` | Layer segment "Jap" has no meaning for a component no longer called a JAP server. Rejected. |
| `JOSYN.Backend.SessionBroker` | Incorrect: the component is no longer in `josyn-backend` and is not a backend library. Rejected. |
| `JOSYN.SessionBroker` | Two-segment form. Explicitly extends the ADR-001 rule: both components at the two-worlds boundary — `JOSYN.JobHost` on the job-developer side, `JOSYN.SessionBroker` on the platform side — use the two-segment form. Accepted. |

**Decision: `JOSYN.SessionBroker`.**

This creates a documented, symmetric pair:

| Namespace | Side | Repo |
|-----------|------|------|
| `JOSYN.JobHost` | Job developer's world | `josyn-job-host` |
| `JOSYN.SessionBroker` | Maintainer's world | `josyn-session-broker` |

The ADR-001 two-segment rule is extended: it applies to both **external consumer packages**
*and* **boundary-crossing EXEs that form the two-worlds interface**. All other
platform-internal packages continue to follow `JOSYN.<Layer>.<Component>`.

---

## Repo structure

```
josyn-session-broker/
├── .local-build/
│   ├── build.cmd
│   ├── pack.cmd
│   └── launch.cmd
├── nuget.config                              ← points to local-packages/
├── JOSYN.SessionBroker.slnx
└── JOSYN.SessionBroker/
    ├── Host.Entrypoint.cs
    ├── Host.Adapters.cs
    ├── Host.Prepare.cs
    ├── AdapterManager.cs
    ├── AdapterProcess.cs
    └── SessionBroker.cs                      ← implements IJosynApplicationProtocol
```

---

## Dependency chain

```
josyn-foundation  ──►  JOSYN.Foundation.JIP
                        JOSYN.Foundation.PropertyBag
                        JOSYN.Foundation.ResultPattern

josyn-jap         ──►  JOSYN.Jap.Contract     (IJosynApplicationProtocol, ErrorReport)

josyn-backend     ──►  JOSYN.Backend.SessionStore
                        JOSYN.Backend.BootstrapConfig
                        JOSYN.Backend.IdentityAdapter.Contract

josyn-commons     ──►  JOSYN.Commons.Log

        all consumed by  ──►  JOSYN.SessionBroker (EXE)
```

All consumed via NuGet. No project references cross the repo boundary. This is identical in
pattern to `josyn-job-host` consuming `josyn-foundation` and `josyn-jap` packages.

---

## What changes

| Area | Change |
|------|--------|
| New repo | `josyn-session-broker` created |
| EXE | `JAPServer.exe` → `SessionBroker.exe` |
| Namespace | `JOSYN.Jap.JAPServer` → `JOSYN.SessionBroker` |
| `josyn-backend` | `josyn-backend-jap-server/` subfolder removed; backend becomes a libraries-only repo (plus `Ticker` and future orchestration EXEs) |
| `josyn-backend` spawn call | `SessionStarter` spawn path updated from `JAPServer.exe` to `SessionBroker.exe` |
| `architecture/naming-conventions.md` | Two-segment rule extended; `JOSYN.SessionBroker` documented as canonical second example alongside `JOSYN.JobHost` |
| `architecture/overview.md` | All references to `JAPServer` / `JOSYN.Jap.JAPServer` updated |
| `repos/josyn-backend/overview.md` | JAPServer entry removed |
| `AGENTS.md` repo table | `josyn-session-broker` added |
| `docs/docs-index.json` | Entry added for new repo |

---

## What does NOT change

| Area | Status |
|------|--------|
| `josyn-jap` repo identity | Unchanged — still the JAP protocol contracts repo |
| `JOSYN.Jap.Contract` | Unchanged — `IJosynApplicationProtocol`, `ErrorReport` unmodified |
| Named pipe protocol (JIP) | Unchanged |
| Session GUID isolation mechanism | Unchanged |
| `josyn-job-host` | Unchanged — `JAPClient` remains `JAPClient` (internal to the library; job developers never see it) |
| Adapter model (ADR-020) | Unchanged — `SessionBroker` spawns adapter EXEs by the same mechanism |
| Historical ADRs | All existing ADRs that reference `JAPServer` are left unchanged as historical records. They describe the state at the time they were written. This ADR supersedes their forward-looking intent. |

---

## Relation to ADR-004

ADR-004 moved `JAPServer` into `josyn-backend` to solve a structural problem: `JAPServer`
needed backend resources (`SessionStore`, `BootstrapConfig`) and had no clean dependency path
to them from `josyn-jap`.

This ADR supersedes ADR-004's location decision. The structural problem is solved differently:
`josyn-session-broker` consumes `josyn-backend`'s library packages via NuGet — the same
mechanism used across every other repo boundary in the platform. The motivation of ADR-004
(clean dependency direction) is preserved; only the implementation changes.

ADR-004's governance note — *"the solution boundary is the primary guard; a project in one
solution must not take a project reference to a project in a different solution"* — is
reinforced, not weakened, by this decision: the repo boundary is now the boundary.

---

## Consequences

- `josyn-session-broker` becomes a first-class repo alongside `josyn-job-host`. The two-worlds
  model is now **visible in the repo topology**: one repo on each side of the boundary.
- `josyn-backend` becomes a cleaner repo: a set of NuGet library packages (SessionStore,
  BootstrapConfig, SessionStarter, JobRegistry, IdentityAdapter.Contract) plus orchestration
  EXEs (Ticker, and future listener/CLI). The per-session boundary EXE no longer lives here.
- The NuGet dependency from `josyn-session-broker` on `josyn-backend` packages is downward and
  intentional: `josyn-backend` is a lower-layer library consumer from `SessionBroker`'s point
  of view. `josyn-backend` does not depend on `josyn-session-broker`.
- Every future maintainer encountering `SessionBroker.exe` in a process list, a log file, or
  a deployment folder has an immediate semantic entry point — no prior architectural context
  required.
