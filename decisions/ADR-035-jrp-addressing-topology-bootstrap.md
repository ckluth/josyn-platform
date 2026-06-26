# ADR-035 — JRP Addressing, Topology, and Bootstrap

**Date:** 2026-06-26
**Status:** Accepted (2026-06-26)

> ADR-033 named the cross-machine seam **JRP** and established that every request carries a
> `JrpTarget { Environment, Machine }`. ADR-034 bound JRP to HTTPS/REST. Neither answered the
> question that sits underneath both: **how does a client decide which server to talk to?** An
> environment is not a single address — it is *n* peer servers with no coordinator. This ADR settles
> the addressing model: what a target *means*, why two classes of verb address differently, where the
> list of servers in an environment lives, and how a client reaches the first one. It introduces no
> new JRP contract type; it decides the *semantics* of the `JrpTarget` the contract already carries.

---

## Context

### What is unspecified after ADR-033 and ADR-034

The JRP contract requires a `JrpTarget` on every request from its first version (ADR-031 DS-2,
ADR-033 T-2), so that cross-machine addressing is a durable contract property rather than a retrofit.
But the contract does not say what a client must *do* to populate that target correctly, nor what a
Gateway does with it on receipt. Concretely, three things are undefined:

- **Resolution** — given that a client wants environment E, how does it arrive at one concrete server?
- **Topology** — what is the authoritative list of servers in environment E, and where does it live?
- **Bootstrap** — how does a client reach its *first* server in E, before it knows any others?

The Gateway host (ADR-033 T-1, ADR-034) cannot be reasoned about correctly without these, because the
meaning of `JrpTarget.Machine` — load-bearing or inert — differs per verb and was never stated.

### The peer model is a deliberate constraint

JOSYN is a set of **peers, not a hierarchy**. Within an environment there are several full, independent
installations, each with its own Gateway. There is **no master node**: no component aggregates,
forwards, fans out, or routes a request on a client's behalf. Each Gateway can answer only for the
environment it belongs to, reading the environment's own database directly.

A precise distinction matters here and is used throughout this ADR:

- A **coordinator / router** would accept a request and *delegate* it to another node. JOSYN has none,
  by design, and this ADR does not introduce one.
- A **directory** merely answers "who are the peers?" It does not route. A directory is compatible
  with the peer model; a router is not.

"No master node" forbids the router. It does **not** forbid a directory.

### One database per environment

There is exactly **one database per environment**, shared by every server in that environment. This
single fact drives the whole addressing model, because it makes almost all data **environment-scoped,
not machine-scoped**: any Gateway in E reads the same database and returns the same answer.

### The placement gap (assumed fixed)

The job-registry currently records *that* a job is registered, but not *on which servers it is
installed*. A job may be installed on one server, several, or all of them. Closing this gap — the
registry carrying a **per-job install set** — is tracked as separate, later work. This ADR **assumes
it fixed** when describing the `start-session` flow, and depends on it only there.

---

## Decision

**Addressing is a client responsibility that terminates in exactly one server. Topology is
platform-owned and environment-local — a single source of truth per environment, with no
replication and therefore no synchronization. The only thing supplied from outside the protocol is a
minimal bootstrap seed, solved by naming convention plus DNS rather than by distributed
configuration.**

### D-1 — Two planes: environment-scoped data verbs vs node-specific execution verbs

JRP verbs divide cleanly by whether the specific server matters:

| Verb class | Addresses | `Machine` load-bearing? | Why |
|---|---|---|---|
| **Environment-scoped data verbs** — the 5 reads + `ChangeJobArgument` | the **environment** | **no** | one DB per env; any Gateway answers identically |
| **Node-specific execution verbs** — `start-session` | a specific **node** | **yes** | a job runs as a physical process on physical hardware |

The cut is **environment-scoped data operation vs node-specific physical execution**, *not* read vs
write — `ChangeJobArgument` is a mutation, but an environment-scoped one (it edits shared-DB data any
peer can serve), so it sits with the reads. For data verbs the machine is *noise*: the client need
only reach **some** live Gateway in the target environment; which one is an availability detail, never
a correctness one. For execution, the machine is the essence of the request — "start this job" is
meaningless without "…on *this* server."

### D-2 — Targets are validated: `Environment` always, `Machine` for execution

The contract keeps `Environment` and `Machine` `required` on every request (no contract change). The
**host** validates and uses them as follows:

- **`Environment` — validated on every verb.** A Gateway belongs to exactly one environment. It
  **rejects** any request whose `Target.Environment` does not match its own configured environment,
  for *all* verbs. A successful response carries the **real serving environment**, never unchecked
  client input — so a DEV Gateway can never read DEV data and return it mislabelled as `PROD`.
- **`Machine` on environment-scoped data verbs — not used for selection.** Because there is one shared
  database per environment, every peer returns the identical answer; the host does **not** filter or
  validate by `Machine`. It merely **echoes** the client-supplied `Machine` into the response DTOs so
  a result is self-locating once more than one installation is reachable. (This matches the throwaway
  `FakeSurfaceAgent` behaviour for the machine field.)
- **`Machine` on `start-session` — validated and load-bearing.** Execution is physical and
  node-specific, so a Gateway **rejects** a launch whose `Target.Machine` is not its own host — it
  never runs a job targeted at another node. Since the machine identifier *is* the node's hostname
  (D-7), the check is simply `Target.Machine == this host`. The concrete rejection
  category/status is an implementation detail of the host phase; the *requirement* is fixed here.

### D-3 — Topology is platform-owned and environment-local (single source of truth, no sync)

The list of servers in an environment lives as **platform data inside that environment's own
database** — one authoritative row set per environment. Consequences:

- There is **one** source of truth for "who is in environment E": E's database. Not three, not *n* —
  one.
- Because there is **no replication**, there is **no cross-store synchronization** to perform and no
  second copy to drift against. Adding or removing a server is a single write to a single database.
  (This is *not* a claim that the row set never lags physical reality — a listed node may be down or a
  decommissioned one may linger; membership maintenance and liveness are a named operational concern,
  see Open Questions. The point is that there is no *second store* to reconcile.)
- The environment is **self-describing**: once a client can reach any Gateway in E, it can read the
  full membership of E.

This is option A of the topology choices considered (see Alternatives). A global, env-independent
topology store (option B) and an external ops/config-store as the owner (option C) were both rejected
because each reintroduces a second source of truth that A avoids entirely.

### D-4 — Placement is platform data in the job-registry (environment-scoped)

"Which servers is job J installed on?" is answered from the per-job install set in the job-registry
(the assumed-fixed placement gap, D-1 context). Because the registry lives in the shared environment
database, this is an ordinary **environment-scoped read**: any Gateway in E answers it identically.
Placement is therefore discovered the same machine-agnostic way as every other read.

### D-5 — `start-session` is inherently two-phase; only the final hop is node-specific

Launching a job decomposes into:

1. **Resolve candidates (environment-scoped read).** The client asks *any* live Gateway in the target
   environment for the install set of job J (D-4). Returns the candidate servers, e.g. `{A, C}`.
2. **Pick one (pure client-side policy).** The client collapses the candidate set to exactly one
   server by *its own* strategy — first, random, round-robin, least-loaded, whatever. The platform
   has no say; there is no coordinator to consult.
3. **Execute on the chosen node (node-specific).** The client issues the real `start-session` call to
   that one server's Gateway, with `JrpTarget.Machine` now naming the picked node. The receiving
   Gateway verifies it *is* that node (D-2) and rejects the launch otherwise; on success the job spawns
   there.

Everything is an environment-scoped read **except the final execution hop**. This is precisely why
`Machine` must exist in the contract even though data verbs do not use it for selection (D-2): step 3
is the one place machine identity is load-bearing.

### D-6 — The bootstrap seed is solved by naming convention + DNS, not by distributed config

D-3 makes the *list* of servers self-serving, but the **first** server cannot be: to read E's
membership a client must already reach one Gateway in E. This irreducible hen-and-egg is resolved by
**convention, not configuration**:

- A **well-known per-environment hostname**, derivable from the environment name by a fixed naming
  rule (e.g. `josyn-dev`, `josyn-int`, `josyn-prod`, per the company DNS pattern).
- The client already knows its environment (its own choice), so it **derives** the seed hostname by
  rule — it stores **no address list**.
- **DNS** resolves that name to **at least one live Gateway** (round-robin or a load-balanced VIP).
  From there the client reads the env-local server-list (D-3) and has full membership.

This keeps two *different* questions with two *different* owners and no duplication:

| Question | Owner |
|---|---|
| "What does `josyn-<env>` resolve to?" (first contact) | **DNS** |
| "Who are all the servers in <env>?" (full membership) | **the environment database** (D-3) |

The seed carries a **rule**, not data — so nothing has to be distributed to clients or kept current
as servers come and go. This is the standard bootstrap pattern (DNS root hints, cluster seed nodes):
full membership discovered *from inside*; a tiny stable seed injected *from outside*. DNS performs
**first-contact resolution only** — it is not a router, so the peer model (no master node) is intact.

### D-7 — Per-node addressing: the machine identifier *is* its resolvable hostname

The two-phase launch (D-5) ends in a call to one *specific* node — but the DNS seed (D-6) deliberately
resolves to *any* node, so it cannot address the chosen one. The gap between "machine `A`" and "`A`'s
concrete endpoint" is closed by **convention, consistent with D-6**:

- `JrpTarget.Machine` **is the node's own DNS-resolvable hostname**. The topology row set (D-3)
  therefore stores pure **identity**; no per-node address data is stored or maintained.
- A node's JRP endpoint is `https://<Machine>:<well-known JRP port>` — scheme and port are a fixed
  platform convention (the per-node analogue of the seed's well-known name). DNS owns name→address.
- HTTPS trust is ordinary TLS: the node's certificate CN/SAN is its hostname; no separate
  cert-identity field is needed.

This keeps the entire topology/bootstrap story under one principle — **convention, not stored
address-data** — and confines drift to DNS (which already owns name→address) instead of duplicating it
into platform data. The seed (D-6) is then just a special "any node" name layered over the same
`https://…:<port>` scheme.

> **Precondition (deployment fact this rests on).** Each server is directly reachable from clients by
> its hostname. If a future deployment places nodes behind proxies/NAT where the identifier is not
> directly addressable, this decision must be revisited — the documented fallback is to store a
> reachable base URI in the topology row (see Alternatives).

---

## Consequences

- **The naming convention becomes architecture.** The per-environment hostname pattern is now a
  contract clients depend on; it must be documented as a deliberate commitment, not left as folklore.
- **DNS gains a named ops obligation.** Whoever owns DNS must keep each `josyn-<env>` name pointing at
  ≥1 live Gateway. This is lighter than maintaining a config-store of full membership, but it is a
  real, named responsibility.
- **Resolution is not liveness.** DNS returns *an* address; that host may be down. Clients must adopt
  a "try the seed, on failure retry / fall back to a known peer" posture. Minor, but real.
- **`JrpTarget` validation is now fully defined** (D-2): the host validates `Environment` on every
  verb and `Machine` on `start-session`, and echoes (without validating) `Machine` on data verbs. A
  Gateway can never serve another environment's data under a mislabelled response. This unblocks the
  Gateway host.
- **Per-node addressing is convention-based** (D-7): `Machine` is a hostname, the JRP scheme/port are
  platform constants, and the topology row set stays pure identity. This commits the platform to
  directly-addressable nodes (see the D-7 precondition).
- **The placement gap is now a named dependency of `start-session` step 1** (D-4/D-5), not an
  incidental TODO. Until it is closed, candidate resolution cannot be served.
- **No JRP contract changes.** `josyn-jrp` is untouched; no package, no version bump (AGENTS.md §8).
  This ADR defines semantics around the existing `JrpTarget`, not new types.

---

## Open Questions

1. **The exact naming + addressing convention.** `josyn-<env>` (the seed, D-6) and the per-node
   `https://<hostname>:<port>` scheme (D-7) are illustrative; the concrete hostname pattern *and* the
   well-known JRP port must align with the company DNS/network scheme. Confirm before clients hard-code
   the derivation.
2. **DNS strategy for liveness (seed).** Round-robin A-records vs a load-balanced VIP vs health-checked
   failover — an ops/deployment decision affecting how robust first-contact is. Out of scope for the
   contract; name it when the first multi-server environment is provisioned.
3. **Topology membership maintenance.** Who writes server rows (D-3), when, and how a stale entry (a
   down or decommissioned node still listed) is detected or tolerated. An operational ownership
   question: the row set is authoritative for *membership*, not a *liveness* oracle (D-3).
4. **Client pick policy (D-5 step 2).** First / random / round-robin / load-aware — left wholly to the
   client. Whether the platform ever *offers* load hints (a read verb) to inform the pick is a future
   question, deliberately not answered here.
5. **A "list peers" / "placement" JRP read verb.** D-3/D-4 make membership and placement readable in
   principle, but no JRP verb exposes either yet. Both are future read verbs gated on the data
   existing. Not in scope for the current host.

---

## Alternatives Considered

- **B — A global, environment-independent topology store** (a distinct database listing all
  environments → servers). Rejected: it is a *second* source of truth that must be kept consistent
  with reality across all environments, reintroducing exactly the synchronization/drift problem that
  D-3's env-local model eliminates.
- **C — Topology owned by the company config-store / external ops configuration**, with JRP having no
  topology concept at all. Rejected as the *owner* of full membership for the same drift reason as B,
  and because it makes the environment non-self-describing. A reduced form of C survives as the
  **seed** mechanism (D-6): the company DNS scheme carries first-contact only, never the full list.
- **A coordinator / master node** that resolves "PROD" to a concrete server and forwards. Rejected on
  principle: it violates the peer model (ADR-033's per-machine Gateway) and creates a single point of
  failure and a routing authority the platform deliberately does not have.
- **Validating `Machine` on environment-scoped data verbs.** Rejected as cost without benefit: under
  one shared environment DB every peer returns the identical answer, so rejecting a "wrong machine"
  read changes no result (D-2). Note this is *only* about the machine dimension — `Environment` **is**
  validated on every verb (D-2) to prevent a Gateway answering for another environment.
- **Storing each node's reachable base URI in the topology row** (instead of deriving it from the
  hostname, D-7). Rejected for the normal case: it reintroduces address-as-data — the maintenance and
  drift burden D-6/D-7 avoid by convention. Retained only as the documented fallback should nodes ever
  become non-directly-addressable (the D-7 precondition).

---

## Relation to Other ADRs

- **ADR-033 (surface three concerns)** — parent. This ADR gives operational meaning to the
  `JrpTarget` that T-2 introduced and honours the per-machine, no-coordinator Gateway model (T-1) by
  forbidding a router while permitting a directory.
- **ADR-034 (JRP HTTP binding)** — sibling. ADR-034 decides *how* JRP is served (Minimal API,
  OpenAPI, Scalar); this ADR decides *which server* a client serves a request to. Together they make
  the Gateway host fully specified.
- **ADR-031 (surface delivery strategy)** — its DS-2 "design the seam for the boundary it will cross"
  is the reason `JrpTarget` exists ahead of need; this ADR confirms that foresight by defining the
  boundary's addressing semantics without a contract change.
- **ADR-010 (environment = installation)** — the "one database per environment" premise (D-3) rests on
  it; this ADR extends it from "an environment is an installation" to "an environment is a set of peer
  servers over one shared database."
- **Job-registry placement work (future)** — hard dependency of D-4/D-5 step 1; `start-session`
  candidate resolution is blocked until the registry carries per-job install sets.

---

## Continuation Brief

> Cold-start orientation for a follow-up session. Read this block first.

**State of play.** *Accepted (2026-06-26).* ADR-033 introduced `JrpTarget { Environment, Machine }`; ADR-034 bound
JRP to HTTPS/REST. The unanswered question — *which server does a client talk to, given an environment
of n peers with no coordinator?* — is settled here.

**The decision in one breath.** Addressing is the client's job and resolves to exactly one server.
There are two planes: **environment-scoped data verbs** (the 5 reads + `ChangeJobArgument`; one DB per
env, any Gateway answers identically; `Machine` echoed but not used for selection) and **node-specific
execution** (`start-session`, runs on physical hardware). The host **validates `Environment` on every
verb** (a Gateway never answers for another environment) and **validates `Machine` on `start-session`**
(it never launches a job aimed at another node). `start-session` is **two-phase**: read the job's
install set from any Gateway (env-scoped), pick one server by client policy, then execute on that node.
**Topology is platform-owned and environment-local** — the server list lives once in each
environment's own database, so there is a single source of truth and **no cross-store synchronization**.
A node's endpoint is **`https://<hostname>:<well-known port>`**: the machine identifier *is* its
hostname (D-7, convention not stored addresses). The unavoidable bootstrap seed (you must reach one
server to learn the rest) is solved by **naming convention + DNS**: a well-known per-environment
hostname, derived from the environment name, resolved by DNS to ≥1 live Gateway. DNS owns
first-contact; the env DB owns full membership. No router, no master node — DNS resolves, it does not
delegate.

**Why it matters.** It defines the meaning of `JrpTarget` per verb, which the Gateway host needs to
behave correctly (validate environment, echo machine on reads, honour and verify it on launch). It
records the peer model and its bootstrap as a *chosen* design, not an accident, and names the
job-registry placement gap as a hard dependency of launch.

**Dependencies / open loops.** The exact DNS naming rule, the well-known JRP port, and the liveness
strategy are ops decisions (Open Questions 1–2). Topology membership maintenance/liveness is an open
operational concern (Open Question 3). A "list peers" / "placement" verb is a future read gated on the
data existing (Open Question 5, D-4).

**Promotion.** Accepted 2026-06-26. The confirmation gate (AGENTS.md §5) still applies to every write
this implies, including cross-referencing this ADR from ADR-033 and ADR-034.

---

## TL;DR

A JRP client must resolve a target down to **one concrete server on its own**, because the platform is
a set of **peers with no coordinator**. **Environment-scoped data verbs** (the 5 reads + argument
change) read one database per environment, so any Gateway answers identically; the host **validates
`Environment` on every verb** (never serving another environment under a mislabelled response) while
`JrpTarget.Machine` is echoed into responses but **not** used to select data. **`start-session`** is
**node-specific** and **two-phase**: read the job's install set from any Gateway, pick one server by
the client's own policy, then launch on that exact node — which the receiving Gateway **verifies it
is** before running. **Topology** — the server list for an environment — is **platform-owned and lives
once in that environment's own database**: single source of truth, no replication, no cross-store sync.
A node's endpoint is **`https://<hostname>:<well-known port>`** — the machine identifier *is* its
hostname (convention, not stored addresses). The irreducible **bootstrap seed** (reach one server to
learn the rest) is solved by **naming convention + DNS**, not distributed config: a well-known
per-environment hostname, derived from the environment name, that DNS resolves to at least one live
Gateway. DNS does first-contact; the environment database does full membership; nothing routes or
delegates. No JRP contract changes.
