# ADR-004 — JAPServer Relocation into josyn-backend

**Date:** 2026-06-01
**Status:** Accepted
**Supersedes:** `proposals/PROP-001-japserver-backend-relocation.md`

---

## Context

JAPServer currently lives in `josyn-jap` as a first-class component, symmetric with
`josyn-job-host`. As the platform matures, JAPServer's protocol implementation must do
real work — reading job arguments from the session store, writing results back, reading
company configuration, and sending execution reports.

All real data sources (`SessionStore`, `CompanyConfig`) are owned by `josyn-backend`.
This creates a structural gap: JAPServer needs backend resources but, as a component of
`josyn-jap`, has no path to them without introducing a reverse or circular dependency.

Four options were evaluated (NuGet shared clients, IPC service, direct DB access,
relocation). The full analysis is preserved in `proposals/PROP-001-japserver-backend-relocation.md`.

---

## Decision

**Move `JOSYN.Jap.JAPServer` (the EXE) from `josyn-jap` into `josyn-backend`.**

`josyn-jap` retains only its two shared packages:

| Package | Role |
|---------|------|
| `JOSYN.Jap.Shared.Contract` | `IJosynApplicationProtocol`, `ErrorReport` — the JAP protocol contract |
| `JOSYN.Jap.Shared.Log` | `LocalLog` — process-local file logger |

`josyn-jap` becomes the **JAP protocol contracts repo** — the single source of truth for
the contract shared by the two parties that speak JAP: `josyn-job-host` (client) and
`josyn-backend` (server, via the relocated JAPServer).

### What does NOT change

- JAPServer.exe is still spawned by backend at session start
- Session isolation by GUID is unchanged
- Named pipe protocol (JIP) is unchanged
- `josyn-job-host` is unchanged
- `josyn-jap` shared packages (`Contract`, `Log`) are unchanged

### Solution structure in josyn-backend

JAPServer receives its **own, separate solution** — it is not merged into the existing
`JOSYN.Backend.SessionStarter.slnx`. Each logical build unit in `josyn-backend` lives in
its own solution file:

```
josyn-backend/
├── JOSYN.Backend.SessionStarter.slnx       ← existing solution (library)
├── JOSYN.Backend.SessionStarter/
│
├── JOSYN.Backend.SessionStarter.Mock.slnx  ← sibling solution (mock EXE for local testing)
├── JOSYN.Backend.SessionStarter.Mock/
│
├── JOSYN.Jap.JAPServer.slnx                ← new solution (relocated EXE)
├── JOSYN.Jap.JAPServer/
│
└── .local-build/
    └── build.cmd                           ← builds ALL solutions
```

The **local-build script builds all solutions** in the repo. No solution is left out of
the automated build.

---

## Consequences

- `josyn-jap` is now a **contracts-only repo** — its identity shifts from "server repo"
  to "protocol contracts repo". The architectural symmetry with `josyn-job-host` breaks;
  this is accepted as the correct trade-off.
- `josyn-backend` gains a NuGet dependency on `JOSYN.Jap.Shared.Contract` and
  `JOSYN.Jap.Shared.Log` (and transitively on `JIP`, `PropertyBag`). This dependency is
  downward and intentional.
- `josyn-backend` becomes a **multi-solution repo**. The local-build script is extended
  to cover all solutions.
- `repos/josyn-jap.md` and `repos/josyn-backend/overview.md` are updated to reflect the new state.

---

## Governance note — backend-repo dependency discipline

`josyn-backend` will grow into the largest and most complex repo in the platform. The risk
of accidental dependency violations is highest here: a shared class library inside the repo
is one project-reference click away, and a wrong reference can silently spoil the
architectural layering.

The solution boundary is the primary guard. Each solution contains only the projects that
belong together by design. A project in one solution **must not** take a project reference
to a project that lives in a different solution within the same repo — it must go via NuGet.

This constraint should be enforced by convention today and by a sanity check criterion once
the repo grows beyond its current stub state.

---

## Decision challenge — objections and rebuttals

This decision was stress-tested by deliberately arguing against it. Six challenges were
raised; each is preserved here with its rebuttal to document why the decision holds.

---

### Challenge 1 — "The problem doesn't exist yet and may never exist in this form"

*Both JAPServer and the backend components it needs are still stubbed. We are solving a
hypothetical.*

**Rebuttal:** The stub phase is exactly the right moment to resolve structural questions —
before real code accumulates in the wrong place. Once JAPServer has a real `GetRawArguments()`
pulling from a database and that code lives in `josyn-jap`, undoing it costs a real migration.
Architecture decisions made under pressure of existing code are always worse than decisions
made while the slate is clean. The problem is not hypothetical — it is a structural
inevitability that is visible now and cheap to address now.

---

### Challenge 2 — "The relocation hides coupling instead of eliminating it"

*A NuGet boundary between repos is replaced by direct project-reference access within the
repo. The intra-repo coupling is harder to see and govern than a package boundary.*

**Rebuttal:** This conflates two different kinds of coupling. A NuGet boundary between
`josyn-jap` and `josyn-backend` would mean `josyn-jap` — a lower-layer repo — taking a
compile-time dependency on `josyn-backend`, the outermost layer. That is a reverse
dependency: an architectural violation by definition. The intra-repo coupling that remains
after relocation connects components that belong together by ownership — JAPServer's
implementation is backend business logic. The governance note does not describe a trap; it
makes a known risk visible and manageable, rather than leaving it hidden behind a repo
boundary that gave false confidence.

---

### Challenge 3 — "Option 2 (IPC) was dismissed too quickly — JIP already exists"

*The platform already has named-pipe IPC. Using JIP for JAPServer-to-backend calls is not
new infrastructure — it is a new use of existing infrastructure.*

**Rebuttal:** JIP is a session-scoped, point-to-point transport: one JAPServer talks to
one job.exe for the duration of one session. Repurposing it for JAPServer-to-backend calls
introduces a fundamentally different topology — many concurrent JAPServer instances each
reaching a single backend service, across session lifetimes. This is not a new use of JIP;
it is a different protocol model that JIP was not designed for. Implementing it would either
require a new multi-client listener model in the backend or break session isolation — one of
the platform's core invariants. The similarity to existing JIP usage is superficial.

---

### Challenge 4 — "josyn-jap's demotion argues for its abolition"

*A repo holding only two small packages doesn't justify its own CI pipeline, release
cadence, and overhead. The relocation makes josyn-jap's existence questionable.*

**Rebuttal:** A repo's value is not proportional to its line count — it is proportional to
the stability and clarity of its boundary. `JOSYN.Jap.Shared.Contract` defines
`IJosynApplicationProtocol`: the contract that two independent processes must agree on to
communicate correctly. That interface is the most load-bearing contract in the platform.
Keeping it in a dedicated repo with its own version history and release cadence ensures that
a breaking change receives its own semver bump, its own changelog, and its own deliberate
review — not a footnote in a backend release. Small and stable is a feature, not a flaw.

---

### Challenge 5 — "`JOSYN.Jap.JAPServer` living in `josyn-backend` is a naming incoherence"

*The namespace says "Jap layer"; the repo says "backend". A developer opening josyn-backend
will find a foreign namespace root sitting inside it.*

**Rebuttal:** The namespace `JOSYN.Jap.JAPServer` reflects what the component *is*: the
server-side implementation of the JAP protocol. That identity does not change because the
source code moved repos. The alternative — renaming it `JOSYN.Backend.JAPServer` — would
be the real incoherence: "Backend" is an orchestration layer, not a protocol layer. A
developer who sees `JOSYN.Jap.JAPServer` in `josyn-backend` understands immediately: "this
is the JAP server executable, and it lives here because it needs backend resources to do its
job." That is a correct and navigable mental model. Renaming to erase the architectural
seam would obscure more than it clarifies.

---

### Challenge 6 — "The multi-solution structure is friction — cross-solution NuGet iteration is painful"

*Pack → push → restore within the same repo is needlessly slow. Using build tooling as a
substitute for discipline imposes a daily cost on developers.*

**Rebuttal:** This scenario does not currently exist in `josyn-backend`. The solutions in
the repo do not share any internal library with each other — each references packages from
*other repos* via NuGet, which is already the established workflow across the entire
platform. There is no intra-repo pack-restore cycle today.

That said, the objection points at a real future decision: if `josyn-backend` ever develops
a shared internal library needed by multiple solutions within the repo, the governance rule
("no cross-solution project references — go via NuGet") would require packing it locally.
At that point the argument does not merely justify the strategy — it *empowers* it: the
friction of the NuGet cycle is precisely the signal that tells a developer "you are crossing
an architectural boundary — be deliberate." A solution boundary with a free project-reference
escape hatch provides zero protection. The local-NuGet discipline turns the boundary into a
real constraint.
