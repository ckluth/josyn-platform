# ADR-031 — josyn-surface: Delivery Strategy (CQRS-lite, Agent Seam, MVP Phasing)

**Date:** 2026-06-21
**Status:** Accepted (2026-06-21)

> **Correction (ADR-033, 2026-06-23):** The "intended durable" Query/Command records this ADR
> places in `JOSYN.Surface.Contracts` relocate to the new contracts-only repo **`josyn-jrp`**
> (`JOSYN.Jrp.Launch` + `JOSYN.Jrp.Surface`); the DS-2 seam invariants now belong to **JRP**. The
> "minimal real platform-resident agent" of DS-5 is named **`JOSYN.Backend.Gateway`** (platform,
> mandatory). The **client-side** seam `ISurfaceAgent` and its `FakeAgent`/`HttpAgent` transports
> stay in `josyn-surface`; the future `HttpAgent` is simply a **JRP client** to the Gateway. The
> DS-4 dev-DB exception is unchanged and still temporary. No behaviour changes. See ADR-033.

> This ADR is a **follow-up to ADR-030** ("josyn-surface: The Human Window into a Headless
> Platform"). ADR-030 fixed *direction* — what the surface is for, who it serves, and twenty
> directional decisions. It deliberately left *how to build it* open. This ADR fills exactly that
> gap: the **internal pattern**, the **mock/abstraction seam**, and the **MVP phasing** by which the
> surface evolves. It refines *how*, never *what*. With one explicit, scoped exception — a
> **temporary, DEV-only exception to D-8/D-17 for phase 1** (DS-4), bounded and removed by construction —
> it leaves ADR-030's twenty decisions (D-1 … D-20) intact. Like ADR-030 it is coarse where the future
> is unknown and precise only where a wrong early choice would be expensive to reverse.

---

## Context

ADR-030 settled the surface's architecture: external shells (Blazor + CLI) and a central aggregator
in a new `josyn-surface` repo, reaching a platform-resident per-machine agent EXE over HTTPS/REST,
delivered in two phases (Reporting Surface → Control Center). What it did **not** settle is the
delivery method: how the first usable thing gets built, what is built for real versus faked, and how
each increment stays a genuine slice of the final architecture rather than a throwaway prototype.

The situation that forces the question is concrete and small. Today there is **one maintainer and one
PoC developer**, both wanting to escape "toolbox chaos" — a scattered set of single-purpose scripts
(ADR-030 § Context). They do not need, and cannot justify standing up, the full three-tier surface
with REST, auth, aggregation, and a web shell before any value is delivered. They need a usable
window *now*, built in a way that does not have to be thrown away *later*.

Three guiding statements from the maintainer frame this ADR:

1. **Clear separation, distinct surface-repo from the start.** The boundary is real from day one,
   not retrofitted. (Reinforces ADR-030 D-13.)
2. **Reusability must not become a burden.** Intermediate working solutions may be abandoned and
   survive only as ideas in later implementations. No sunk-cost fallacy on cheap code.
3. **CQRS must stay a logical thing, not an academic architecture.** No framework, no ceremony —
   a conceptual split, nothing more.

These three pull against each other if taken naïvely: "distinct repo + reuse everything" wants a full
backbone up front, while "evolve slowly + abandon cheap code" wants almost nothing built. This ADR
resolves the tension by being precise about *what is expensive to retrofit* (build it) versus *what
is cheap to add or discard later* (fake it, or defer it).

---

## Decisions

### DS-1 — CQRS-lite: a logical split, not an architecture

The surface organises its operations as two conceptual buckets:

- **Queries** — read operations. An immutable request record in, a `Result<T>` out.
- **Commands** — write/control operations. An immutable request record in, a `Result<T>` out.

That is the entire commitment. Explicitly **rejected** (the "academic CQRS" baggage):

- No MediatR or any dispatch framework; no reflection-based handler registry.
- No separate read and write data stores; one store per environment (ADR-030 C-4) serves both.
- No event sourcing, no command bus, no eventual consistency, no projections.

A query or command is a plain immutable record; its handler is a **static function returning
`Result`** (coding-standards § Result pattern; ADR-030 C-3). The split exists to keep read and write
honestly separate (ADR-030 D-2) and to make the eventual REST mapping natural (GET = query,
POST = command) — not to import a methodology. "CQRS" here is shorthand for *"we name reads and
writes separately,"* nothing more.

### DS-2 — The seam is the agent, not the transport

ADR-030's real boundary (D-8 API-mediated, D-16 cross-machine REST, D-17 platform-resident agent) is
**surface → per-machine agent**. The thing that is abstracted — and therefore the thing that is
*mocked* in early phases — is the **agent**, never the HTTP transport.

A single seam expresses this:

```
Shell  →  Query/Command record  →  ISurfaceAgent  →  Result<T>
                                     ├─ phase 1: FakeAgent  (in-process; reads dev DB directly)
                                     └─ phase 2: HttpAgent → real platform-resident agent (D-16/D-17)
```

Swapping `FakeAgent` for `HttpAgent` changes **no** query, command, handler, or shell code —
*provided the seam is designed for REST realities from the start* (the invariants below). This is
why the seam — not the transport — is the load-bearing abstraction. "Mock HTTP" was the wrong framing;
"mock the agent behind a stable seam" is the right one, because it is HTTP-shaped only in phase 2 and
the contract above it never knows the difference.

This seam **is** the boundary ADR-030 D-20 demanded between the idiomatic-web exception zone and the
functional-first core: shell and host above the seam may be idiomatic; records, handlers, and the
`Result` discipline below it are functional-first C#.

**Seam invariants (designed from MVP-1, not retrofitted at phase 2).** The "changes nothing above the
seam" guarantee only holds if the contract anticipates the network boundary it will later cross.
Therefore, from the first query, the seam obeys:

- **Async by shape.** `ISurfaceAgent` is asynchronous and cancellable, even though `FakeAgent` answers
  synchronously. An in-process method that later sprouts `async`/`CancellationToken` *is* a breaking
  change above the seam — so it is async from day 1.
- **Wire-safe records.** Every request and response record must be serialisable and free of in-process-only
  types. No table/DB shapes cross the seam (see DS-4); records are transport-safe DTOs.
- **Identity in the contract.** Every request carries **target environment + machine** identity, and an
  **actor** field (enforcement deferred — D-9), so cross-machine/cross-environment targeting and audit are
  not retrofitted later (ADR-030 FR-16, NFR-3).
- **A stable error taxonomy, not a bare `Result.Error`.** The `Result` failure surface must name the
  categories REST will introduce — *not-found*, *unavailable/unreachable*, *unauthorized*, *timeout/cancelled*,
  *partial* (for future aggregation) — so phase-2 transport failures map onto existing cases rather than
  forcing new ones. `Result<T>` crossing the seam is **wire-safe**: it carries no exceptions, call stacks,
  or internal diagnostics (those stay in-process / in logs).
- **Bounded reads.** List queries (e.g. sessions) take a range/page bound from MVP-1, so REST does not later
  introduce pagination as a contract change.

### DS-3 — Build the durable layer for real; fake or defer everything else

The only assets built "for real" early are the ones expensive to retrofit. Everything else is faked
(phase 1) or deferred (later), and may be discarded without regret (maintainer statement 2).

| Asset | Status | Rationale |
|-------|--------|-----------|
| Query / Command record contracts | **Intended durable** (subject to transport validation) | Cheap to write; expensive to retrofit. Phase-2 REST inherits them **only if** they obey the DS-2 seam invariants (wire-safe, identity-bearing, no DB shapes). |
| `Result<T>` return shape on every handler | **Intended durable** | The one omission that would force rewriting every handler later. Must carry the wire-safe error taxonomy (DS-2), not a bare error. |
| `ISurfaceAgent` seam | **Durable** | Swaps FakeAgent → HttpAgent with zero change above it; it *is* the D-20 line. Exact shape (one entry vs two) is DSQ-1. |
| One CLI shell | **Mostly durable** | Power-ops shell (ADR-030 D-7, R1–R2); Blazor later reuses the same contracts. |
| `FakeAgent` + its phase-1 SQL + DB↔DTO mapping | **Throwaway** | Reads the dev DB directly and maps to durable DTOs; abandoned the moment the real agent lands. |
| HTTP host, serialization | **Deferred** | Adding them changes no handler if the seam invariants hold. |
| Auth / RBAC, audit | **Deferred** (contract fields designed-in) | Enforcement later (ADR-030 D-9, D-10); the actor/target identity *fields* exist from MVP-1 (DS-2). |
| Central aggregator, machine registry | **Deferred** | Needed only once there is more than one machine to reach (ADR-030 D-4, D-15, D-18). |
| Blazor web shell | **Deferred** | Broad-reach shell (R4–R6) follows once the read path is proven. |

The discipline in one line: **build the contracts, the `Result` shape, the seam, and one shell;
fake the agent; defer the rest.** "Intended durable" carries an obligation: a record is only durable if
it passes the DS-2 seam invariants — otherwise it is a DB-view in disguise and will churn at phase 2.

### DS-4 — Phase 1 FakeAgent: a temporary, scoped, DEV-only exception

In phase 1 the `FakeAgent` lives in the `josyn-surface` repo and **reads the dev DB directly**. This is
a conscious, explicitly **scoped exception** to ADR-030 D-8 (API-mediated, not direct SQL) and D-17
(store access should be platform-resident) — not a claim that those decisions don't apply. It is
accepted under strict bounds:

- **DEV only, read-only.** The exception covers reads against a single **DEV** installation. No write
  path, no int/prod connection, no arbitrary/runtime connection strings — `FakeAgent` is compiled and
  configured for one DEV installation only (ADR-010: environment = installation, not a runtime switch).
- **Disposable scaffolding, not smuggled architecture.** Abandoned wholesale when the real
  platform-resident agent and `HttpAgent` arrive — no migration, no sunk cost (maintainer statement 2).
- **Keeps the surface repo clean and standalone from day 1** (maintainer statement 1, ADR-030 D-13)
  without first standing up the agent EXE, REST host, and cross-machine plumbing that ADR-030 NFR-7/NFR-8
  warn against front-loading.

**Containment is a rule, not a hope.** "Below the seam" only contains the breach if no DB shape escapes
it. Therefore: **no DB table/row/join shape may cross `ISurfaceAgent`.** `FakeAgent` maps its raw SQL
results to the durable, transport-safe DTOs (DS-2) *inside* itself; that mapping is throwaway (DS-3). A
durable record that happens to mirror a table is a DB-view in disguise and breaks the phase-2 "inherit
verbatim" guarantee — so each durable response type is justified on agent/API terms, independently of the
current schema.

The exception must be named as such in the `FakeAgent` source and tracked, with its removal tied to the
arrival of the real agent, so it is never mistaken for the intended design.

### DS-5 — MVP phasing: read-only first, command later

Increments follow ADR-030's two-phase delivery and the "safe by construction" instinct: the Reporting
Surface (read-only) precedes any mutation.

- **MVP-1 — read-only.** Queries `GetRecentSessions` and `GetErrorDetail`, run from a local CLI against
  the dev DB via `FakeAgent`. Replaces the *capabilities* (not the code — ADR-030 D-11) of
  `get-session-report` and `get-error-report`. Safe by construction: no mutations.
- **MVP-2 — first command, gated on a real platform-side agent.** `RetriggerSession`, the first write.
  The surface repo **never links `SessionLauncher` or any platform launch code.** A command that spawns a
  session crosses ADR-030's control boundary (C-2) and is authorised for the **platform-resident agent**
  (D-17, D-19), not the external surface. Therefore MVP-2 is **gated on a minimal real platform-resident
  agent EXE in `josyn-backend`**: the agent constructs the `SessionStartRequest` and calls `SessionLauncher`
  directly (D-19, ADR-017B-01); the surface only sends the `RetriggerSession` command across the seam.
  Transport may still be deferred — the agent can be invoked in-process behind `ISurfaceAgent` *while
  residing in `josyn-backend`* — but placement is correct from MVP-2: launch code stays platform-side. This
  closes, rather than extends, the DS-4 exception (which is read-only and never covered launching).
- **Later increments.** Swap `FakeAgent` → `HttpAgent` and complete the real platform-resident agent's
  REST surface (D-16/D-17); migrate the remaining read handlers platform-side; add the Blazor shell,
  aggregator, machine registry, and auth/audit enforcement as their need becomes real.

**Command envelope (designed at MVP-2, enforced later).** Because MVP-2 introduces the first mutation,
its command record carries — from the start — the fields that are expensive to retrofit into a write
contract: **actor identity**, **target environment + machine**, and a **correlation id** for the audit
trail. Confirmation gating and audit *enforcement* remain deferred (ADR-030 D-9, D-10), but the contract
does not omit the concerns. This is the minimum that keeps "design auth/audit now, enforce later" honest
without over-building a full RBAC envelope.

Each MVP is a **contract-honest** slice of the final architecture: the same identity-bearing record +
handler + wire-safe `Result` that later phases expose over REST. MVP-1 is thinner than the final topology
— it omits the aggregator, registry, REST, and the multi-machine axis (ADR-030 D-4/D-6/D-15/D-16) — so it
is a **local DEV reporting precursor**, not a slice through the full topology. What makes it non-throwaway
is the *contract*, which is final-shaped (DS-2). What is thrown away (per DS-4) is the *scaffolding behind
the seam*, never the contract above it.

---

## Consequences

- The `josyn-surface` repo is created **at the start of MVP-1** (ADR-030 D-13, maintainer statement 1),
  holding the durable contracts, the `ISurfaceAgent` seam, the CLI shell, and the disposable `FakeAgent`.
- The platform (`josyn-backend`) gains **nothing at MVP-1**, but **MVP-2 triggers a minimal
  platform-resident agent EXE** there (ADR-030 D-17/D-19): launch code never enters the surface repo.
  The cross-machine REST API (D-16) is still deferred — the agent may be invoked in-process at MVP-2 — but
  its placement is correct from the first command.
- A **temporary, DEV-only, read-only exception** to D-8/D-17 exists in phase 1 (DS-4): direct dev-DB access
  from the surface repo, contained by the no-DB-shape-crosses-the-seam rule, and removed when the real
  agent lands. It never covers launching.
- The maintainer gains usable value early — two toolbox report tools replaced by MVP-1 — while every
  early brick (contracts, `Result`, seam, shell) is a permanent part of the final surface.
- ADR-030's twenty decisions remain intact **except** the scoped, temporary DS-4 exception to D-8/D-17.
  This ADR otherwise only resolves the *delivery* gap they left open.

---

## Open Questions

These are deliberately left for the moment the relevant increment begins, not resolved speculatively now.

### DSQ-1 — Seam shape: one dispatcher or two?
DS-2 names a single `ISurfaceAgent` seam. CQRS-lite (DS-1) might instead expose two thin entry points
(`Query(...)` / `Command(...)`) to make the read/write asymmetry visible at the call site (ADR-030 D-2).
Both keep the swap property of DS-2. Decide when MVP-1's first query is written; it is a small,
reversible choice.

### DSQ-2 — Where MVP-2's command handler lives ✅ Closed (see DS-5)
**Decision:** MVP-2 is **gated on a minimal real platform-resident agent EXE in `josyn-backend`**; the
surface repo never links `SessionLauncher` or any launch code. Spawning a session crosses ADR-030's
control boundary (C-2) and is the platform-resident agent's job (D-17, D-19). Transport may be deferred
(in-process invocation behind `ISurfaceAgent`), but placement is correct from the first command. This
closes — rather than extends — the read-only DS-4 exception.

### DSQ-3 — Cross-machine layer name (inherited from ADR-030)
ADR-030's single open loop — the *name* of the cross-machine REST layer — remains open and is **not**
needed until `HttpAgent` exists. Listed here only so it is not lost.

---

## Relation to Other ADRs

- **ADR-030 (josyn-surface)** — parent. This ADR fills its open "how to build it" gap. It honours
  D-1 … D-20 **except** the scoped, temporary DS-4 exception to **D-8/D-17** (DEV-only, read-only,
  phase-1, removed when the real agent lands). In particular it operationalises **D-2** (read/write
  separation → DS-1), **D-7/D-8** (CLI-first, API-mediated → DS-2/DS-5), **D-13** (own repo → DS-4),
  **D-17/D-19** (platform-resident agent owns launch → MVP-2 gating, DS-5/DSQ-2), **D-9/D-10** (auth/audit
  designed-in → command envelope, DS-5), and **D-20** (idiomatic/functional boundary → the seam in DS-2).
- **ADR-017B-01 (SessionLauncher relocation)** — supplies the launch primitive the **platform-resident
  agent** (not the surface) reuses for MVP-2.
- **ADR-010 (environment separation)** — phase 1 targets a single **DEV** installation only; the direct-DB
  exception (DS-4) is confined there (no runtime connection-string switching) and never touches int/prod.
- **coding-standards.md (Result pattern, static-first, immutability)** — governs every durable handler
  and record below the seam (ADR-030 C-3).

---

## Continuation Brief

> Cold-start orientation for a follow-up session. Read this block first.

**State of play.** This ADR is *Proposed*. It resolves the delivery-method gap ADR-030 left open. Five
decisions are settled: **DS-1** CQRS-lite (logical split, no framework, no separate stores),
**DS-2** the seam is the agent not the transport (`ISurfaceAgent`, swap FakeAgent→HttpAgent with zero
change above — *guarded by seam invariants*: async, wire-safe identity-bearing records, named error
taxonomy, bounded reads), **DS-3** build only the durable layer (contracts, `Result`, seam, one shell),
fake or defer the rest, **DS-4** phase-1 `FakeAgent` reads a single DEV DB directly as a temporary,
read-only, scoped exception to D-8/D-17, with **no DB shape allowed to cross the seam**, **DS-5** read-only
MVP-1 first (`GetRecentSessions`, `GetErrorDetail`); the first command (`RetriggerSession`, MVP-2) is
**gated on a minimal real platform-resident agent in `josyn-backend`** — the surface never links launch
code — and its command record carries actor/target/correlation fields from day 1 (enforcement deferred).
Open: **DSQ-1** (seam shape, one entry vs two) and **DSQ-3** (inherited cross-machine name); **DSQ-2** is
now closed.

**The decided delivery model in one breath.** A distinct `josyn-surface` repo from day 1 holds immutable
Query/Command records, a `Result`-returning handler per operation, a single `ISurfaceAgent` seam, and a
CLI shell. Phase 1 fakes the agent (`FakeAgent`, in-process, direct dev-DB read — disposable). Later
phases swap in `HttpAgent` talking to the real platform-resident agent over REST (ADR-030 D-16/D-17),
migrate handlers platform-side, and add the web shell, aggregator, registry, and auth as need becomes
real. Only the scaffolding behind the seam is ever thrown away; every slice above it is permanent.

**On acceptance, the work (not yet started — none of this exists):**
1. Create the `josyn-surface` repo with the durable layer: identity-bearing, wire-safe Query/Command
   records, the `ISurfaceAgent` seam (async, named error taxonomy), `FakeAgent` (DEV-only, read-only,
   with internal DB↔DTO mapping), and a CLI shell.
2. Build **MVP-1**: `GetRecentSessions` + `GetErrorDetail`, read-only, local CLI against a single DEV DB.
3. Resolve **DSQ-1** (seam shape) when the first query is written.
4. For **MVP-2**, stand up the minimal platform-resident agent EXE in `josyn-backend` (it owns the
   `SessionLauncher` call); the surface only sends `RetriggerSession`.
5. Update **ROADMAP.md** and **docs/docs-index.json** to register `josyn-surface` and MVP-1.

**Promotion.** Accepted 2026-06-21 (with ADR-030). Implementation begins with the scoped MVP-1
work list above.
