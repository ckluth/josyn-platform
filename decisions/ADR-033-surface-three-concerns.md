# ADR-033 — josyn-surface is Three Concerns: the JRP Gateway, JRP Contracts, and Edge Clients

**Date:** 2026-06-23
**Status:** Proposed

> This ADR does not add a feature. It **re-conceptualises** what "josyn-surface" *is*. ADR-030
> defined the surface as a single product that, by its own admission, *"straddles the boundary"*
> between the platform and the external world. ADR-032 then folded the Listener into the
> "surface agent", giving that component the **sole network launch path** — a core platform
> capability now wearing a surface badge. This ADR resolves the resulting concern-overload by
> splitting "surface" into the **three** distinct things it has actually become, each landing
> cleanly on one side of the platform boundary, and it fixes the vocabulary (**JRP**, **Gateway**)
> that the split requires. It changes naming and ownership, not behaviour.

---

## Context

### The straddle was there from the start

ADR-030 settled the surface's direction but left one seam unresolved in its own decisions:

- **D-3** — the surface is **external tooling**, alongside the platform like the toolbox.
- **D-17** — the per-machine **agent** is a **platform component, resident in `josyn-backend`**.

ADR-030 reconciled these only by admitting the contradiction out loud (SQ-2):

> *"josyn-surface as a product straddles the boundary — external consumers talking to a
> platform-resident agent."*

A single named product that is simultaneously *external* and *platform-resident* is not a clean
boundary; it is a deferred decision. It was tolerable only while the "agent" did nothing but
*serve* the surface.

### ADR-032 tipped it over

ADR-032 removed the standalone Listener and folded `start-session` into the surface agent. That
changed the agent's **nature**, not just its verb count. Network-initiated session launch is a
**core orchestration capability** — it would exist even if no human ever opened a dashboard. After
ADR-032 the agent owns the **sole network launch path on every machine** and is therefore
**mandatory platform infrastructure**. A platform-core responsibility now lives inside a component
named *surface*. This is precisely the *"collapse two concerns into one identity"* drift the
platform repeatedly guards against (ADR-024; ADR-005 no-duplication).

### The questions this forces

Three questions, asked by the maintainer, have no clean answer **while "surface" means one thing**:

1. **Is the surface part of the backend?** *Partly* — and "partly" is the disease, not the answer.
2. **Is the surface optional?** *It depends which half you mean* — which proves the overload.
3. **Is the surface one thing, or three?**

This ADR answers: **three**.

---

## Decision

**"Surface" is three distinct concerns, each wholly on one side of the platform boundary. No
single component straddles the line.**

```
   josyn-surface  (clients — OPTIONAL, edge)        josyn-backend
   ┌───────────────────────────────┐            ┌────────────────────────────┐
   │ JOSYN.Surface.Cli              │            │ JOSYN.Backend.Gateway      │
   │ JOSYN.Surface.Web   (deferred) │── JRP ───► │  hosts JRP, reads stores,  │
   │ JOSYN.Surface.SessionClient    │  (HTTPS/   │  performs control,         │
   └───────────────┬────────────────┘   REST)    │  owns start-session        │  MANDATORY
                   │                              └────────────┬───────────────┘
                   │            josyn-jrp (SEAM — contracts only)             │
                   └──────────►  JOSYN.Jrp.Launch   (core/orchestration) ◄────┘
                                 JOSYN.Jrp.Surface  (read/control)
```

### T-1 — The platform edge is a backend component: the **Gateway** (mandatory)

The per-machine, platform-resident host is named **`JOSYN.Backend.Gateway`**. It is the rename of
today's `JOSYN.Backend.SurfaceAgent`. It is **platform, not surface**:

- It is **resident in `josyn-backend`** and present on **every** machine (ADR-030 D-17).
- It **hosts the JRP API** (T-2), reads the local stores and bootstrap INI, and performs control.
- It **owns `start-session`** — the sole network launch path (ADR-032) — constructing a
  `SessionStartRequest` and calling `SessionLauncher` directly (ADR-030 D-19, ADR-017B-01).
- It is **not optional**: removing it removes network-initiated launch from the machine.

Calling this thing "surface" was the root of the overload. It is the platform's **gateway** onto
each machine — the network entry point through which remote callers reach local platform
mechanisms. "Surface" no longer names it.

> **Why "Gateway" despite ADR-025.** ADR-025 *rejected* "Gateway" for `SessionBroker` because
> *"'Gateway' carries strong network/API infrastructure connotations; invites confusion with API
> gateways."* That objection was correct for a **same-machine, per-session** boundary EXE. This
> component is the opposite: a genuine **cross-machine HTTPS/REST** endpoint. The very connotation
> ADR-025 steered away from the broker is the one that **fits** here. Using "Gateway" now is
> consistent with ADR-025, not in tension with it.

### T-2 — The seam is a named protocol: **JRP** (JOSYN Remote Protocol)

The cross-machine layer ADR-030 D-16 left unnamed (its single open naming loop, SQ-4) is named
**JRP — JOSYN Remote Protocol**. The name encodes the boundary axis the platform already uses:

| Protocol | Boundary | Transport |
|----------|----------|-----------|
| **JAP** — JOSYN **Application** Protocol | same machine, per session | named pipes |
| **JIP** — JOSYN **Interprocess** Protocol | same machine, process ↔ process | named pipes |
| **JRP** — JOSYN **Remote** Protocol | **cross machine** | **HTTPS / REST** |

`I`nterprocess (machine-bound) versus `R`emote (cross-machine) is the distinction that earns the
name. JRP is **wholly distinct from JIP** (ADR-030 D-16): distinct transport, distinct contract,
distinct concerns. JIP/JAP remain same-machine IPC and are untouched.

**The JRP contracts live in their own contracts-only repo, `josyn-jrp`** — parallel to `josyn-jap`
(contracts, no EXE). They carry **two sub-contract families**, kept separate because ADR-032
(Decision 3) requires it:

| Package | Concern | Lifecycle |
|---------|---------|-----------|
| **`JOSYN.Jrp.Launch`** | `start-session` request/response (job name, base64 args, allocated GUID) | **Small, stable, core/orchestration.** Exists for its own sake; a launch consumer needs nothing else. |
| **`JOSYN.Jrp.Surface`** | read queries + control commands (the dashboard verbs) | **Churning, surface-facing.** Evolves with the human window. |

This is the key refinement the listener-kill demanded: the Gateway has **two contract surfaces with
different owners and lifecycles** sharing one host. They are kept apart in the contract layer, not
merged. "Surface" survives here only as a **concern qualifier** (`JOSYN.Jrp.Surface`), never as a
component name.

The JRP contracts are the durable artefact ADR-031 DS-2/DS-3 called *"intended durable."* They
obey the DS-2 **seam invariants** unchanged: async by shape, wire-safe and identity-bearing records,
a named error taxonomy, bounded reads. No DB shape crosses them (ADR-031 DS-4).

### T-3 — The surface is the optional **edge clients**

"Surface" now means exactly the **human-facing edge**: the optional clients, and the read/control
concern they serve. The `josyn-surface` repo **keeps its name but holds clients only**:

| Client | Binds |
|--------|-------|
| `JOSYN.Surface.Cli` (exists) | `JOSYN.Jrp.Surface` (+ `JOSYN.Jrp.Launch` when it triggers) |
| `JOSYN.Surface.Web` (deferred, Blazor) | `JOSYN.Jrp.Surface` |
| `JOSYN.Surface.SessionClient` (deferred, ADR-032) | **only** `JOSYN.Jrp.Launch` |

There may be **many** clients; there may be **zero**. The surface concern is **optional** — the
platform runs headless without it, exactly as ADR-030 C-1 describes today. A launch-only consumer
binds only `JOSYN.Jrp.Launch` and never swallows the dashboard contract (ADR-032 Decision 4).

The client-side transport seam `ISurfaceAgent` (ADR-031 DS-2) is a **client concern** and stays in
`josyn-surface`. Its `FakeAgent`/`HttpAgent` implementations are client transports; the future
`HttpAgent` is simply **a JRP client** speaking to the Gateway. (Whether `ISurfaceAgent` is itself
renamed — e.g. to a JRP-client seam — is deferred; see Open Questions.)

---

## The three questions, answered

| Question | Answer under the three-concern model |
|----------|--------------------------------------|
| **Is the surface part of the backend?** | The **Gateway** is (platform). The **clients** are not (edge). The **JRP contracts** are the seam between them. **No component straddles the boundary anymore** — the D-3/D-17 contradiction is dissolved, not deferred. |
| **Is the surface optional?** | The **surface concern (clients)** is optional — many, or none. The **Gateway** is **not** optional; ADR-032 made it the sole network launch path. The old "is it optional?" had no answer only because one word meant both halves. |
| **Is it one thing, or three?** | **Three**: Gateway (platform), JRP / `josyn-jrp` (seam), Surface clients (edge). |

---

## What changes in code (decided here; applied as the surface work continues)

The surface code is young (MVP-2b is not yet committed per the 2026-06-22 session summary), so these
renames are cheap and carry no migration cost. They are **decisions**, applied incrementally — not a
demand for an immediate big-bang refactor.

| Today | Becomes | Side |
|-------|---------|------|
| `JOSYN.Backend.SurfaceAgent` (EXE) | **`JOSYN.Backend.Gateway`** | platform |
| `SurfaceCommandHandler` (platform write path) | moves into `JOSYN.Backend.Gateway` | platform |
| wire records in `JOSYN.Surface.Contracts` | move to **`JOSYN.Jrp.Launch`** / **`JOSYN.Jrp.Surface`** (repo `josyn-jrp`) | seam |
| `JOSYN.Surface.Cli` | unchanged name; consumes `JOSYN.Jrp.*` | edge |
| `ISurfaceAgent`, `FakeSurfaceAgent`, `CompositeSurfaceAgent` | stay client-side in `josyn-surface`; throwaway DS-4 scaffolding unaffected; future `HttpAgent` is a JRP client | edge |

The DS-4 phase-1 exception (FakeAgent reading the dev DB directly) is **unchanged and still
temporary**; it is replaced when the JRP `HttpAgent` talks to the Gateway, exactly as ADR-031 DS-5
describes. Nothing above `ISurfaceAgent` changes.

---

## Supersedes & Updates

The human reviews each change. None are code-behaviour changes; they are decision-record, naming,
and roadmap corrections.

| Source | Current state | Change |
|--------|---------------|--------|
| **ADR-030** D-3 / D-17 / SQ-2 | "the product straddles the boundary; external consumers talking to a platform-resident agent." | The straddle is **resolved** by the three-concern split. The "agent" is renamed **Gateway** and is unambiguously **platform** (T-1). Only the *clients* are "external" (D-3); the Gateway is platform (D-17) — now stated as three separate things, not one straddling one. |
| **ADR-030** D-16 / SQ-4 | the cross-machine HTTPS/REST layer is *"its own name — deliberately not JIP-anything"*, left open. | **Closed.** The layer is named **JRP — JOSYN Remote Protocol**. ADR-030's single remaining open naming loop is retired. |
| **ADR-031** DS-2/DS-3 | "intended durable" Query/Command records in `JOSYN.Surface.Contracts`; seam is `ISurfaceAgent`. | The durable wire records relocate to **`josyn-jrp`** (`JOSYN.Jrp.Launch` / `.Surface`). The seam invariants are unchanged and now belong to JRP. `ISurfaceAgent` stays a **client** seam. |
| **ADR-032** Decisions 2–4 | "the surface agent gains a `start-session` verb"; "launch sub-contract small, stable, separate"; `SessionClient` binds only the launch sub-contract. | "surface agent" → **`JOSYN.Backend.Gateway`**. The launch sub-contract **is** `JOSYN.Jrp.Launch`; the read/control surface is `JOSYN.Jrp.Surface`. `SessionClient` binds `JOSYN.Jrp.Launch`. No behaviour change. |
| **ROADMAP.md** | surface work tracked as one product. | Register **JRP**, the **`josyn-jrp`** contracts repo, and the **Gateway** rename. Note the surface concern (clients) as optional/edge. |
| **`architecture/naming-conventions.md`** | lists JAP, JIP. | Add **JRP** (JOSYN Remote Protocol; cross-machine; `josyn-jrp` repo, `JOSYN.Jrp.*`) and the `JOSYN.Backend.Gateway` component. |
| **`architecture/overview.md` / `repos/`** | describe `josyn-surface` and a "surface agent". | Reflect the three concerns: Gateway (backend), `josyn-jrp` (seam), `josyn-surface` (clients). |
| **`docs/docs-index.json`** | — | Register ADR-033 and `josyn-jrp`. |

---

## Rationale

**Why split rather than just rename the agent?**
A rename alone (agent → Gateway) fixes the *name* but not the *model*: "surface" would still mean a
straddling product. The split fixes the model — three concerns, three locations, one clean boundary
— and the names follow from it. The listener-kill proved the agent is platform; the split makes that
structural, not nominal.

**Why a named protocol (JRP) and a contracts repo (`josyn-jrp`)?**
The platform already expresses every cross-boundary contract as a named protocol in its own
contracts repo (`josyn-jap`, `josyn-adapter-contracts`). The cross-machine boundary deserves the
same treatment, not an anonymous "REST API." A named seam is what lets clients and the Gateway
depend on the *contract*, never on each other.

**Why keep `JOSYN.Jrp.Launch` and `JOSYN.Jrp.Surface` apart?**
Because they have different owners and lifecycles. Launch is core orchestration (stable, exists with
zero clients); Surface is the human window (churns). Merging them would re-couple a security-sensitive
launch path to UI churn — the exact failure ADR-032 Decision 3 forbids. Separate packages let a thin
client bind only what it needs (ADR-032 Decision 4) **by construction**, not by discipline.

**Why does the surface keep its repo name?**
Because, once the Gateway and JRP are extracted, `josyn-surface` finally *is* what its name claims:
the human-facing clients at the edge. The name stops over-reaching the moment the other two concerns
leave it.

---

## Consequences

- A new contracts-only repo **`josyn-jrp`** is anticipated, holding `JOSYN.Jrp.Launch` and
  `JOSYN.Jrp.Surface`. It parallels `josyn-jap`; it produces **no EXE**.
- `josyn-backend` gains **`JOSYN.Backend.Gateway`** (the renamed surface agent). It is mandatory,
  per-machine, and owns the JRP host and `start-session`.
- `josyn-surface` becomes a **clients-only** repo. The surface concern is explicitly **optional/edge**.
- ADR-030's last open naming loop (the cross-machine layer name) is **closed** as **JRP**.
- ADR-031's "intended durable" contracts find their permanent home (`josyn-jrp`); the DS-4 FakeAgent
  exception is unchanged and still temporary.
- The boundary the platform cares about is restored: **no component is simultaneously external and
  platform-resident.** Gateway = platform; JRP = seam; Surface = edge.
- All affected documents (above) are corrected by the human at review time.

---

## Open Questions

1. **`josyn-jrp` packaging granularity.** One package with two namespaces, or two distinct NuGet
   packages (`JOSYN.Jrp.Launch`, `JOSYN.Jrp.Surface`)? Two packages let a thin client reference only
   `Launch`. Recommended: **two packages**; confirm when `josyn-jrp` is created.
2. **Client-side seam name.** Does `ISurfaceAgent` (ADR-031 DS-2) keep its name, or become a
   JRP-client seam (e.g. `IJrpClient`)? It is a client concern; deferred until the `HttpAgent` phase.
3. **Rename timing.** Apply the `SurfaceAgent → Gateway` and contract-relocation renames now (code is
   pre-commit, cheap) or fold them into the next surface increment? Recommended: **now**, before MVP-2b
   is committed, to avoid renaming committed history.
4. **Does the Gateway host non-surface remote verbs later?** As "the machine's JRP gateway," it may in
   future carry remote platform verbs unrelated to the human surface. Left open; the three-concern
   model already accommodates it (such verbs would be new JRP sub-contracts, not "surface").

---

## Relation to Other ADRs

- **ADR-030 (josyn-surface)** — parent. This ADR resolves its D-3/D-17 straddle and closes its
  D-16/SQ-4 naming loop (JRP). The "agent" of D-17 is the **Gateway**.
- **ADR-031 (surface delivery strategy)** — its durable contracts relocate to `josyn-jrp`; its seam
  invariants become JRP's; `ISurfaceAgent`, `FakeAgent`, and the DS-4 exception remain client-side and
  unchanged in behaviour.
- **ADR-032 (no standalone Listener)** — its "surface agent" is the **Gateway**; its required small,
  stable launch sub-contract **is** `JOSYN.Jrp.Launch`, kept separate from `JOSYN.Jrp.Surface`;
  `SessionClient` binds only `JOSYN.Jrp.Launch`.
- **ADR-025 (session-broker)** — its rejection of "Gateway" for the *same-machine* broker is honoured;
  "Gateway" is used here for the genuine *cross-machine* network endpoint, the case ADR-025's objection
  did not cover.
- **ADR-023 (adapter-contracts repo) / `josyn-jap`** — precedents for a named-protocol, contracts-only
  repo. `josyn-jrp` follows the same pattern.
- **ADR-017B-01 (SessionLauncher relocation) / ADR-019 / ADR-029** — supply the inner launch mechanism
  the Gateway reuses for `start-session`; unaffected.
- **JIP / JAP** — untouched. JRP is a third, cross-machine protocol; it does not extend or rename them.

---

## Continuation Brief

> Cold-start orientation for a follow-up session. Read this block first.

**State of play.** *Proposed.* This ADR re-conceptualises "josyn-surface" as **three concerns**,
triggered by ADR-032 folding the Listener into the so-called "surface agent" — which made that
component own the **sole network launch path** and exposed the long-standing D-3/D-17 straddle in
ADR-030.

**The decision in one breath.** Surface is not one thing but three, each cleanly on one side of the
platform boundary: **(1) the Gateway** — `JOSYN.Backend.Gateway`, a *platform*, mandatory, per-machine
EXE in `josyn-backend` that hosts JRP, reads stores, and owns `start-session` (the renamed surface
agent); **(2) JRP** — *JOSYN Remote Protocol*, the cross-machine HTTPS/REST seam (vs JIP =
Interprocess, machine-bound), with contracts in a new contracts-only repo **`josyn-jrp`** split into
`JOSYN.Jrp.Launch` (small/stable/core) and `JOSYN.Jrp.Surface` (read/control/churning); **(3) the
Surface** — the *optional* edge clients in `josyn-surface` (`JOSYN.Surface.Cli` / `.Web` /
`.SessionClient`), each binding only the JRP sub-contracts it needs.

**Vocabulary locked this session.** JRP (not JEP — homophone with JAP in spoken German; not JOP —
reads like "Job"). Gateway (Steward/Warden rejected as unfamiliar in the German tech domain; Node
rejected as topological, under-signalling the trust boundary). The Gateway name is consistent with
ADR-025, which only rejected "Gateway" for the same-machine broker.

**Work on acceptance (none of this is done yet):**
1. Create `josyn-jrp` (contracts only): `JOSYN.Jrp.Launch`, `JOSYN.Jrp.Surface` — two packages
   (Open Q 1).
2. Rename `JOSYN.Backend.SurfaceAgent` → `JOSYN.Backend.Gateway`; move `SurfaceCommandHandler` into it.
3. Relocate the durable wire records from `JOSYN.Surface.Contracts` into the `JOSYN.Jrp.*` packages;
   keep `ISurfaceAgent`/`FakeAgent`/`CompositeSurfaceAgent` client-side.
4. Update ADR-030/031/032 cross-references, `architecture/naming-conventions.md`,
   `architecture/overview.md`, `repos/`, `ROADMAP.md`, and `docs/docs-index.json` per Supersedes &
   Updates.
5. Resolve Open Questions 1–3 when the renames land (recommended: before MVP-2b is committed).

**Promotion.** Awaiting maintainer review. The confirmation gate (AGENTS.md §5) applies to every
write the work list above implies.
