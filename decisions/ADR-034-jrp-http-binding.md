# ADR-034 — JRP HTTP Binding: Hosting Stack and Contract Authority

**Date:** 2026-06-26
**Status:** Accepted (2026-06-26)

> ADR-033 named the cross-machine seam **JRP** and fixed its transport as **HTTPS/REST**, but left
> the concrete HTTP binding undefined. This ADR settles the parts of that gap that are *architectural*:
> the **hosting and documentation stack** for the **Gateway host** (the platform-resident JRP EXE,
> ADR-033 T-1) and the **contract-authority** question the chosen stack forces — **where does the JRP
> contract live?** The *mechanical* remainder — the concrete per-verb route, HTTP verb, status code,
> and serialisation table — is deliberately **delegated to the host's `IEndpoint` implementations**,
> not frozen here (see Context). This ADR changes no JRP contract; it decides how that contract is
> *served* and *documented*.

---

## Context

### What is unspecified after ADR-033

ADR-033 T-2 states JRP's transport is HTTPS/REST and that the Gateway "hosts the JRP API." It does
not say:

- how each JRP request/response record maps to an HTTP route, verb, and status code;
- what serves the human-readable API documentation;
- whether the surface clients hand-write their HTTP client or generate it.

This ADR answers the **architectural** questions — the hosting/documentation stack and where the
contract lives (the second and third points, plus the *form* of the mapping in the first). The
**concrete per-verb route/verb/status table** (the first point's detail) is intentionally **not frozen
in this ADR**: it is owned by the host's `IEndpoint` classes and reviewed at the host phase, because
freezing a route table in a decision record would duplicate authority that already lives in the
endpoint code and `josyn-jrp` types. What this ADR fixes is the *stack*, the *contract authority*, and
the *rule* that the mapping reference `josyn-jrp` types directly — not the literal routes.

### The inspiration — a sibling project's REST stack

A separate company project documented a clean Minimal-API stack (`REST-Sample.md`): ASP.NET Core
Minimal API with an `IEndpoint`-per-class pattern, `Microsoft.AspNetCore.OpenApi` generating a live
`/openapi/v1.json`, **Scalar** rendering that spec as an interactive docs UI, API versioning, and
**Kiota** generating a typed client from the spec. Its stated core principle:

> *The API contract lives once in the endpoint code; documentation, UI, and client SDK are derived
> from it automatically — no drift between implementation and documentation is possible.*

That principle is excellent for a project whose contract has no other home. **JOSYN is not that
project.**

### The collision

JOSYN already has a contract home: **`josyn-jrp`** (ADR-033 T-2), a contracts-only repo of
hand-owned `sealed record`s split into `JOSYN.Jrp.Launch` and `JOSYN.Jrp.Surface`. Both the Gateway
host and the surface clients bind the **same** `JOSYN.Jrp.*` NuGet packages; the records *are* the
contract (see the "contract lives through the records" model — ADR-033 T-2, ADR-031 DS-2).

The REST-Sample principle and the JOSYN model each declare a **single source of truth**, and they are
different sources:

| | Source of truth | Client |
|---|---|---|
| **REST-Sample** | endpoint code → OpenAPI spec | **generated** (Kiota) from the spec |
| **JOSYN** | `josyn-jrp` records (NuGet) | **binds the same records**; client is hand-written |

If JOSYN adopted Kiota, the generated client would materialise a **second, parallel set of DTOs**
from the OpenAPI spec — duplicating the exact types `josyn-jrp` already owns. That violates ADR-005
(no duplication) and dissolves the "contract is the records" model that ADR-033/ADR-031 deliberately
established. Kiota's value proposition — "don't hand-write the client" — is largely **moot** here:
the client already shares the contract types via NuGet. The over-the-wire `HttpAgent` (deferred,
ADR-033 T-3) needs only a thin `HttpClient` that serialises the shared records, not generated copies
of them.

---

## Decision

**Adopt the hosting and documentation half of the REST-Sample stack; keep `josyn-jrp` as the sole
contract authority; reject Kiota and generated clients.**

### D-1 — `josyn-jrp` remains the single source of truth for the JRP contract

The `JOSYN.Jrp.*` records are authoritative. The HTTP layer is a **binding over** them, never a
**definer of** them. No JRP type is introduced, shaped, or owned by the host or the OpenAPI spec.

### D-2 — Hosting: ASP.NET Core Minimal API + `IEndpoint`-per-class, explicitly registered

The Gateway host serves JRP via ASP.NET Core Minimal API. Each JRP verb is one `IEndpoint`
implementation in its own file (one concern per file — matches the §9 code-generation principles:
short methods, named groups, no bloated controllers). Endpoints are **registered explicitly** by a
hand-written static list/registration method — **not** auto-discovered by assembly reflection scan and
**not** wired through a DI container. This is deliberate: JOSYN's house style is static-first, no
DI-by-default (AGENTS.md / coding-standards), so the endpoint set is a visible, greppable enumeration
rather than runtime magic. The full surface is readable from one file.

### D-3 — OpenAPI is a *projection* of the contract, not a *source* of it

The host uses `Microsoft.AspNetCore.OpenApi` to generate `/openapi/v1.json` at runtime. Endpoints
reference the existing `josyn-jrp` types directly (e.g. `.Produces<StartSessionResponse>()`), so the
spec **documents** the contract that already exists in `josyn-jrp`. The spec is a read-only view; it
never feeds back into type definitions. This keeps docs **closely tracking** the contract **without**
creating a competing authority — the very risk D-1 guards against.

> The "no drift" property is real but **bounded**: the spec cannot drift from the *types* it
> projects, because it is generated from them. It *can* still under-describe behaviour an endpoint
> author forgot to annotate (a missing `.Produces<…>()`, an unmapped status). The host phase therefore
> carries an explicit **acceptance check**: every JRP verb has exactly one registered endpoint that
> references its `JOSYN.Jrp.*` types, and the `JrpError`→status metadata (D-7) is reviewed. The
> guarantee is "no type drift", not "no human omission."

### D-4 — Scalar for the human-facing API docs UI

Scalar renders the same `/openapi/v1.json` as an interactive browser UI for developers. No separate
model, no extra config — it reads what the endpoints already describe. (Replaces Swagger UI.)

### D-5 — API versioning on the JRP routes

JRP routes are versioned (`/v1/...`). JRP is a durable wire contract crossing machine and (later)
release boundaries; versioned paths are a contract property from day one, consistent with the
"design the seam for the boundary it will cross" stance (ADR-031 DS-2, ADR-033 T-2).

### D-6 — Reject Kiota and generated clients

No client is generated from the OpenAPI spec. The deferred `HttpAgent` (ADR-033 T-3) is a
**hand-written** JRP client: an `ISurfaceAgent` implementation that serialises the shared
`JOSYN.Jrp.*` records over `HttpClient`. The OpenAPI spec and Scalar remain useful for documentation,
exploration, and ad-hoc/third-party callers — they are simply not the basis of JOSYN's own client.

### D-7 — `JrpError` → HTTP status mapping is owned by the host edge

The `JrpErrorCategory` taxonomy maps to HTTP status codes at the host's outbound edge (e.g.
`NotFound` → 404), and the reverse mapping lives in the future `HttpAgent`, so the named taxonomy —
not raw HTTP codes — remains the contract above `ISurfaceAgent`. The exact table is an implementation
detail of the host phase, not a contract change.

---

## Rationale

**Why adopt the stack at all?** It is idiomatic modern ASP.NET Core, the `IEndpoint` pattern matches
JOSYN's one-concern-per-file standard exactly, and OpenAPI+Scalar deliver live API docs that track the
contract types automatically (no *type* drift — see D-3's bounded guarantee) — directly answering the
"JRP HTTP binding is undefined" gap ADR-033 left open.

**Why keep `josyn-jrp` authoritative instead of the endpoint code?** Because JOSYN already paid for a
named, versioned, NuGet-distributed contract repo precisely so clients and host depend on the
*contract*, never on each other (ADR-033 T-2, ADR-023/`josyn-jap` precedent). Letting endpoint code +
OpenAPI become the authority would relocate the contract into the host — re-coupling the seam to one
implementation and abandoning the contracts-only-repo pattern the platform uses everywhere else.

**Why reject Kiota specifically?** Its entire benefit is generating client types when none exist. Here
they exist, shared by NuGet. Generating them again creates duplicate DTOs (ADR-005 violation) and two
sources of truth for one contract. A hand-written `HttpAgent` over the shared records is *less* code
and *more* consistent, not a regression.

**Why OpenAPI/Scalar if not for client generation?** Documentation and exploration are valuable on
their own: a live, always-current human view of the JRP surface, an interactive test console, and a
machine-readable spec for any third party who is *not* a JOSYN-internal client and therefore cannot
bind `josyn-jrp` packages. Projection, not generation.

---

## Consequences

- The Gateway host introduces the **first ASP.NET Core web stack** into `josyn-backend`
  (`Microsoft.AspNetCore.OpenApi`, Scalar, API-versioning packages). This is a new dependency class
  for that repo and is accepted deliberately here.
- `josyn-jrp` stays contracts-only and unchanged; no new package, no version bump (AGENTS.md §8).
- The deferred `HttpAgent` is scoped as a thin hand-written JRP client over shared packages — **not** a
  generated SDK. This narrows ADR-033 T-3's "future HttpAgent" to a concrete shape.
- OpenAPI/Scalar endpoints (`/openapi/v1.json`, Scalar UI) become part of the Gateway host's surface;
  their exposure/security posture is a host-phase concern (see Open Questions).
- The "contract is the records" model (ADR-033 T-2) is reaffirmed and protected against erosion by a
  generated-client back door.

---

## Open Questions

1. **Spec/UI exposure.** Are `/openapi/v1.json` and Scalar served in all environments or only
   non-production? They describe a security-sensitive launch path. Default proposal: enabled in
   dev/test, gated in production. Decide in the host phase.
2. **Versioning scheme detail.** URL-segment (`/v1/...`) is assumed; confirm vs header-based when the
   first `v2` need is real. Low cost to defer.
3. **Hosting model.** Console host vs `Microsoft.Extensions.Hosting` worker / Windows Service — a
   deployment decision for the host phase, not a contract decision; out of scope here.
4. **`JrpError` → status table.** The concrete mapping is implementation detail (D-7); ratify the table
   during the host build.

---

## Relation to Other ADRs

- **ADR-033 (surface three concerns)** — parent. This ADR implements its T-2 "HTTPS/REST" promise by
  defining the binding, and concretises its T-3 "future HttpAgent" as a hand-written JRP client. It
  also resolves ADR-033 Open Question 4's spirit by confirming OpenAPI/Scalar as the documentation
  surface for the JRP host.
- **ADR-031 (surface delivery strategy)** — its DS-2 seam invariants are preserved: the binding is
  shaped for the network boundary; `ISurfaceAgent` stays the client seam above the wire.
- **ADR-005 (no duplication)** — the decisive constraint against Kiota-generated DTOs.
- **ADR-023 / `josyn-jap`** — precedent for a named-protocol, contracts-only repo as the authority;
  this ADR keeps JRP faithful to that pattern rather than relocating the contract into endpoint code.
- **ADR-035 (JRP addressing, topology, bootstrap)** — sibling. ADR-034 decides *how* JRP is served
  (Minimal API, OpenAPI, Scalar); ADR-035 decides *which server* a client serves a request to (peer
  model, environment-scoped reads vs node-specific launch, env-local topology, DNS seed). Together the
  two fully specify the Gateway host.

---

## Continuation Brief

> Cold-start orientation for a follow-up session. Read this block first.

**State of play.** *Accepted (2026-06-26).* ADR-033 named JRP and set its transport to HTTPS/REST but left the HTTP
binding undefined. A sibling project's `REST-Sample.md` (ASP.NET Core Minimal API + `IEndpoint` +
`Microsoft.AspNetCore.OpenApi` + Scalar + versioning + Kiota) inspired this decision.

**The decision in one breath.** Adopt the **hosting + documentation** half of that stack for the
Gateway host (Minimal API, `IEndpoint`-per-class, OpenAPI, Scalar, versioning). Keep **`josyn-jrp` as
the single contract source of truth**: OpenAPI is a *projection* of the existing records, never their
definer. **Reject Kiota** — generated client DTOs would duplicate the `josyn-jrp` contract (ADR-005),
creating two sources of truth. The deferred `HttpAgent` is therefore a **hand-written** JRP client
over the shared `JOSYN.Jrp.*` packages.

**Why it matters.** It protects the "contract is the records" model (ADR-033 T-2) while still buying
live, interactive API docs that track the contract types. It also concretises the shape of the future
`HttpAgent` and unblocks the Gateway host phase (Part C item #1).

**Promotion.** Accepted 2026-06-26. The confirmation gate (AGENTS.md §5) still applies to every write
the Gateway-host work implies, including cross-referencing this ADR from ADR-033.
