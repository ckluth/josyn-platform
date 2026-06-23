# ADR-032 — No Standalone Listener: Session-Start Folds into the Surface Agent

**Date:** 2026-06-23
**Status:** Accepted

> **Correction (ADR-033, 2026-06-23):** "the surface agent" throughout this ADR is renamed
> **`JOSYN.Backend.Gateway`** (platform, mandatory — no longer "surface"). The required small,
> stable **launch sub-contract** (Decision 3) **is** **`JOSYN.Jrp.Launch`**, kept separate from the
> read/control surface **`JOSYN.Jrp.Surface`**; both are verbs of **JRP — JOSYN Remote Protocol**
> hosted by the Gateway. `SessionClient` (Decision 4) binds only `JOSYN.Jrp.Launch`. No behaviour
> changes. See ADR-033.

---

## Context

Across several earlier ADRs the platform anticipated a dedicated orchestrator EXE called the
**Listener**:

- **ADR-017B-01** named `listener` as one of four planned orchestrators (`listener`, `ticker`,
  `cli`, `workflow-runner`) and noted that a REST-based orchestrator would receive arguments as a
  base64 string in its JSON body.
- **ADR-024** carried `JOSYN.Backend.Listener.exe` in the `Orchestrators\Listener\` deployment
  slot and listed the Listener among the orchestrators that call `SessionLauncher`.
- **`guides/session-launch.md`** drew a "REST listener (future)" box calling `SessionLauncher`.
- **ROADMAP** M3 lists the Listener as the remaining piece of the scheduling milestone, with a
  planned package `JOSYN.Backend.Listener` (`josyn-backend-listener`).

The Listener's single defined capability was: *receive a start-job request over REST → construct a
`SessionStartRequest` → call `SessionLauncher.LaunchSession()`.* This is exactly the inner launch
mechanism that `TimeScheduler` already uses (the `SessionStartRequest` + base64-argument +
`SessionLauncher` line, per ADR-017B-01 and the session-launch guide).

**What changed.** ADR-030 (josyn-surface) introduced a **platform-resident per-machine agent EXE**
that:

- is a **distinct sibling EXE in `josyn-backend`** (D-17),
- exposes a **cross-machine HTTPS/REST API**, wholly distinct from JIP (D-16), and
- **acts as its own orchestrator** for trigger / re-trigger: it *"constructs a `SessionStartRequest`
  and calls `SessionLauncher` directly"* — the sanctioned reuse point — explicitly **rejecting**
  delegation to the Listener's REST endpoint to avoid a second hop and a dependency on the Listener
  being deployed (D-19, SQ-6).

ADR-030 **SQ-2** settled only the *inverse* question — *"is the agent the Listener?"* → no, keep
them separate, the Listener stays a thin orchestrator. It never re-justified why a **separate**
Listener still earns its place **after** the agent gained launch-control over REST. That is the gap
this ADR closes.

### The overlap

The Listener's `start-session` and the agent's trigger/re-trigger are the **same capability**
(network request → `SessionStartRequest` → `SessionLauncher`), over the **same transport family**
(cross-machine HTTPS/REST, D-16), from an EXE that **already exists in the same repo** and is
**resident on every machine**. A separate Listener would be a second resident REST host doing a
strict subset of what the agent already does — duplication that ADR-005 (no duplication) and the
platform's minimalism principles push against.

### The only reasons a separate Listener could still be justified

Three justifications were considered, each of which would have kept the Listener alive:

1. **Surface-optional deployment** — network job-launch needed on machines where the surface/agent
   is *not* installed.
2. **Minimal trusted surface** — a job-start-only endpoint with a smaller attack surface than the
   full agent.
3. **Contract decoupling** — a programmatic launch client that must not be coupled to the agent's
   UI-driven, churning contract.

The maintainer resolved all three:

- The **surface agent is an integral part of the backend** — always present. Justification 1 does
  not hold.
- There is **no security concern** specific to a job-start-only endpoint. Justification 2 does not
  hold.
- The only real concern is that a **launch client must stay small** — it must not carry the whole
  surface stack. This is a **client-packaging concern, not a server-identity concern**.

A small client does **not** require a separate server. Client granularity is decoupled from server
identity: one REST host (the agent) can serve many clients of different sizes, each binding only the
verbs it needs. Solving a client-size problem by standing up a second resident REST EXE would be
duplication for nothing.

---

## Decision

### 1. There is no standalone Listener

`JOSYN.Backend.Listener` is **not built**. The Listener is removed from the platform's planned
component set. The capability it was to provide — network-initiated session launch — is provided by
the surface agent.

### 2. `start-session` is a verb on the surface agent's REST API

The agent's cross-machine HTTPS/REST API (ADR-030 D-16) gains a **`start-session`** verb. It does
exactly what every orchestrator does and what the Listener was to do:

- accept the job name and base64-encoded arguments in the request body (the REST transport
  convention from ADR-017B-01 and the session-launch guide),
- construct a `SessionStartRequest` with `Arguments` already base64 (no re-encoding), and
- call `SessionLauncher.LaunchSession()` — the same inner mechanism `TimeScheduler` uses (the
  "template" reuse point), per D-19.

This is the agent playing the **orchestrator role** for one more verb — consistent with how D-19
already framed trigger / re-trigger. It is not a new component; it is one endpoint on an existing
one.

### 3. The launch sub-contract is kept small and stable, separate from the read/control surface

So that a thin client can bind only what it needs without inheriting UI-driven churn, the agent's
**launch verb(s)** are organised as a **small, stable sub-contract** — distinct request/response
records, separate from the surface's read and control (dashboard) verbs, which evolve on their own
schedule. Keeping the launch contract stable and isolated is the discipline that makes a thin client
possible; it is the server-side counterpart to Decision 4.

### 4. Client granularity is a client concern — deferred to the second building step

The originally-planned **`SessionClient`** remains a future, separate building step (out of scope
here). When built, it binds **only** the agent's launch sub-contract (Decision 3) — a small contract
and a small implementation. It does **not** carry the surface read/control contracts.

The recommended shape is **layered, not merged**: a small core launch client, with an optional
fuller surface client built on the same transport. A launch-only consumer must never be forced to
swallow the whole surface client. The exact client topology (one client vs. `SessionClient` +
`SurfaceClient`) is decided in that later step, not here.

### 5. Hosting, framework and dual-mode questions belong to the agent, not here

The questions of *which* HTTP host the REST API runs on, and how it presents service vs. console
modes, are **the agent's concern** and are already governed by **ADR-030 D-20** (the web shell and
REST host are an accepted idiomatic-web exception zone, with a stated functional-first boundary for
domain/access logic). This ADR introduces **no new host or framework decision**. The earlier
Listener-specific debate about hand-rolled `HttpListener` vs. ASP.NET is moot — there is no
Listener.

---

## Supersedes & Updates

This ADR is the controlling decision on the Listener. The following references are superseded or
must be updated; the human reviews each change.

| Source | Current state | Change |
|--------|---------------|--------|
| **ADR-030** D-17 / SQ-2 | "The Listener stays a thin orchestrator; the agent is a distinct sibling EXE, not the Listener." | Superseded in part: there is no Listener at all. The agent is not "distinct from the Listener" — it is the **sole** network launch path. The agent-vs-Listener separation rationale is retired. |
| **ADR-024** | Lists `Listener` as an orchestrator; shows `Orchestrators\Listener\JOSYN.Backend.Listener.exe (future)` in the deployment layout. | Remove the Listener from the orchestrator list and the deployment layout. Ticker, TimeScheduler, WorkflowRunner, CLI are unaffected. |
| **ADR-017B-01** | "Four orchestrator executables are planned: `listener`, `ticker`, `cli`, `workflow-runner`." | The planned set drops `listener`. The base64-argument convention it established stands and now applies to the agent's `start-session` verb. |
| **`guides/session-launch.md`** | "REST listener (future)" box calls `SessionLauncher`; closing note describes "a REST listener" receiving base64 arguments. | Rename/replace the "REST listener" box with the surface agent's `start-session` verb. The base64 transport rule is unchanged and now described against the agent. |
| **ROADMAP** | M3 row "Scheduling — listener, ticker, time-based triggers"; "implement `JOSYN.Backend.Listener` (the remaining piece)"; package `JOSYN.Backend.Listener` (`josyn-backend-listener`). | Drop the Listener from M3 and from the planned package list. M3's remaining work is re-scoped (the agent's `start-session` verb lands with the surface work, not as a standalone Listener). |
| **`docs/docs-index.json`** | — | Register ADR-032. |

---

## Rationale

**Why fold rather than keep a thin Listener "for separation"?**
Separation is only worth a second resident EXE when it buys deployment independence, a smaller
trusted surface, or contract isolation. The maintainer ruled out the first two outright, and the
third is achievable on the client side (Decisions 3–4). With every justification removed, a separate
Listener is pure duplication of a capability the agent already exposes on every machine.

**Why is a small client still possible without a small server?**
Because a client binds a *subset* of an API. The agent's launch verb is given its own stable
sub-contract (Decision 3); a thin client binds only that. The launch consumer is insulated from
dashboard/UI churn by contract organisation, not by a separate process.

**Why not repurpose the name "Listener" for the agent's verb?**
The agent already owns the network edge under its own (still-to-be-named) cross-machine API
(ADR-030's open naming loop). Introducing "Listener" as an alias for one of its verbs would
resurrect the very component this ADR retires and invite drift. `start-session` is a verb, not a
component.

---

## Consequences

- **No `JOSYN.Backend.Listener` package or EXE** is produced. The `Orchestrators\Listener\`
  deployment slot is not used. No `deploy-maintainer.ps1` publish step for the Listener is added.
- The **surface agent** gains a `start-session` verb on its REST API and an obligation to keep the
  **launch sub-contract small, stable, and separate** from its read/control verbs.
- The **`SessionClient`** (second building step) is scoped to the launch sub-contract only and is
  designed as a layered client — small core launch client, optional fuller surface client.
- Several documents are updated per **Supersedes & Updates** above. None of these are code changes;
  they are decision-record and roadmap corrections that the human reviews.
- The earlier Listener HTTP-host / framework question (`HttpListener` vs. ASP.NET) is closed as moot:
  hosting is the agent's concern under ADR-030 D-20.

---

## Open Questions

1. **Agent REST API name** — still ADR-030's single open loop (the cross-machine REST layer's name).
   Unaffected by this ADR; the `start-session` verb lives under whatever that API is eventually
   named.
2. **Client topology** — one client vs. `SessionClient` + `SurfaceClient`, and exactly where the
   launch sub-contract records live. Deferred to the second building step.
3. **Launch sub-contract shape** — the precise request/response records for `start-session` (job
   name, base64 arguments, and what a successful response carries — e.g. the allocated session
   GUID). Decided when the agent verb is implemented.

---

## Relation to Other ADRs

- **ADR-030** (josyn-surface): provides the agent (D-17), its cross-machine REST API (D-16), the
  direct-`SessionLauncher` launch role (D-19, SQ-6), and the REST-host exception zone (D-20) into
  which `start-session` lands. This ADR supersedes the parts of D-17/SQ-2 that presumed a coexisting
  Listener.
- **ADR-031** (surface delivery strategy): the `start-session` verb is part of the agent's REST
  surface that later phases stand up (the `FakeAgent` → `HttpAgent` seam). No conflict.
- **ADR-024** (Ticker): the Ticker and its orchestrator targets (TimeScheduler, WorkflowRunner, CLI)
  are unaffected; only the Listener is removed from its orchestrator list and deployment layout.
- **ADR-017B-01** (Session-Starter Relocation): the planned orchestrator set drops `listener`; the
  base64-argument transport convention stands and now applies to the agent's `start-session` verb.
- **ADR-029** (TimeScheduler Evaluation Strategy): TimeScheduler remains the canonical example of the
  inner launch mechanism (`SessionStartRequest` + `SessionLauncher`) that `start-session` reuses.
