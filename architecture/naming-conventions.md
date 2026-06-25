# Naming Conventions

## Vocabulary Map

| Word | Scope | Usage |
|------|-------|-------|
| **Platform** | The entire JOSYN ecosystem | Umbrella term — all repos, all layers |
| **Backend** | The scheduler and session-orchestration layer | `josyn-backend` repo, `JOSYN.Backend.*` namespace |
| **JAP** | The JAP protocol contracts layer — defines the session protocol shared between the two worlds | `josyn-jap` repo, `JOSYN.Jap.*` namespace |
| **JRP** | The JRP protocol contracts layer — the **cross-machine** (Remote) HTTPS/REST protocol; distinct from JIP (Interprocess, machine-bound) (ADR-033) | `josyn-jrp` repo, `JOSYN.Jrp.*` namespace (`Launch`, `Surface`) |
| **Gateway** | The platform-resident, per-machine **command library** that implements JRP write commands and owns `start-session` (ADR-032/033). Currently a library (`JOSYN.Backend.Gateway`); a Gateway EXE/service host is deferred to the `HttpAgent`/`SessionClient` phase. | `josyn-backend` repo, `JOSYN.Backend.Gateway` |
| **Surface** | The **optional**, human-facing edge **clients** (CLI, web, session client) — the consumer side of JRP (ADR-030/033) | `josyn-surface` repo, `JOSYN.Surface.*` namespace |
| **Session Broker** | The per-session boundary EXE — brokers between the backend world and the job developer's world | `josyn-session-broker` repo, `JOSYN.SessionBroker` namespace |
| **Foundation** | Cross-cutting infrastructure primitives | `josyn-foundation` repo, `JOSYN.Foundation.*` namespace |
| **Job Host** | The job execution runtime | `josyn-job-host` repo, `JOSYN.JobHost` namespace |
| **Commons** | Generic utility helpers — domain-agnostic, open for growth | `josyn-commons` repo, `JOSYN.Commons.*` namespace |
| **Adapter** | The boundary layer between the platform and company adapter EXEs | `josyn-adapter-contracts` repo, `JOSYN.Adapter.*` namespace |

> **Note:** "JAP" does **not** mean "backend" in this codebase. `josyn-jap` holds the JAP protocol contracts — the shared interface between the two worlds. The scheduler role belongs exclusively to `josyn-backend`. The per-session boundary EXE is `josyn-session-broker`. See [../decisions/ADR-001-platform-naming.md](../decisions/ADR-001-platform-naming.md) and [../decisions/ADR-025-session-broker.md](../decisions/ADR-025-session-broker.md).

> **Decision:** "Platform" was chosen as the umbrella word because "JAP" already carries a specific architectural meaning (the JAP session server). See [../decisions/ADR-001-platform-naming.md](../decisions/ADR-001-platform-naming.md).

---

## Repository Names

```
josyn-foundation        ← infrastructure primitives
josyn-jap               ← JAP protocol contracts (contracts-only; no EXE)
josyn-jrp               ← JRP protocol contracts (cross-machine; contracts-only; no EXE) (ADR-033)
josyn-job-host          ← job execution runtime  
josyn-session-broker    ← per-session boundary EXE (brokers between the two worlds)
josyn-backend           ← scheduler and session-orchestration layer; NuGet library packages
josyn-commons           ← generic utility helpers — domain-agnostic, open for growth
josyn-adapter-contracts ← JIP protocol contracts for company adapter EXEs (ADR-023)
josyn-surface           ← optional human-facing edge clients (CLI/web/session) (ADR-030/033)
josyn-platform          ← this repo; architecture + decisions + docs
```

Pattern: `josyn-<layer>` — all lowercase, hyphen-separated.

The absence of "jap" in `josyn-job-host` (despite the `JOSYN.Jap.*` layer packages it depends on) is **deliberate**: it signals that job executables are architectural outsiders — decoupled consumers of the protocol, not internal components of the scheduler. This reasoning is extended to the namespace: `JOSYN.JobHost` (not `JOSYN.Jap.JobHost`) hides the internal "Jap" protocol layer from the developer who just wants to author a job.

> **Intentional naming exception — documented rule:**  
> Packages at the **two-worlds boundary** use the two-segment form `JOSYN.<Component>` without a layer segment. This applies to:  
> - Packages whose primary audience is *external job authors* (consumers outside the JOSYN platform): `JOSYN.JobHost` is the canonical example.  
> - The platform-side boundary EXE: `JOSYN.SessionBroker` (ADR-025) — the symmetric counterpart to `JOSYN.JobHost`.  
> All other platform-internal packages follow the three-segment `JOSYN.<Layer>.<Component>` pattern without exception. See [../decisions/ADR-001-platform-naming.md](../decisions/ADR-001-platform-naming.md) and [../decisions/ADR-025-session-broker.md](../decisions/ADR-025-session-broker.md).

---

## Namespace Conventions

```
JOSYN
├── Foundation                     ← josyn-foundation packages
│   ├── ResultPattern              ← error-as-value primitives
│   ├── PropertyBag                ← record serialization
│   └── JIP                        ← named pipe transport
│       └── Jip                    ← JIP convention layer
├── Jap                            ← josyn-jap (contracts only — no EXE)
│   └── Contract                   ← IJosynApplicationProtocol, ErrorReport
├── Jrp                            ← josyn-jrp (contracts only — no EXE) (ADR-033)
│   ├── Launch                     ← start-session request/response (core, stable)
│   └── Surface                    ← read queries + control commands (churning)
├── Adapter                        ← josyn-adapter-contracts (ADR-023)
│   ├── ConfigurationAdapter
│   │   └── Contract               ← IConfigurationAdapter
│   └── IdentityAdapter
│       └── Contract               ← IIdentityAdapter
├── SessionBroker                  ← josyn-session-broker (two-segment — boundary EXE, see ADR-025)
├── JobHost                        ← josyn-job-host (two-segment — consumer-facing API, see ADR-001)
│   └── Attributes                 ← [JobEntryPoint], [BeforeJobEntryPoint], etc.
├── Backend                        ← josyn-backend
│   ├── SessionStarter             ← session lifecycle rendezvous (stub)
│   └── Gateway                    ← platform-resident command library; GatewayCommandHandler owns write path (ADR-032/033); EXE host deferred
└── Commons                        ← josyn-commons (utility satellite — never referenced by Foundation)
    └── ...                        ← packages added as helpers accumulate
```

Pattern: `JOSYN.<Layer>.<Component>[.<Subcomponent>]` — with two intentional exceptions:  
`JOSYN.JobHost` and `JOSYN.SessionBroker` use the two-segment form — see the rule under *Repository Names* above.

---

## Project / Assembly Names

Assembly names match their namespace root exactly:

| Assembly | Namespace | Repo |
|----------|-----------|------|
| `JOSYN.Foundation.ResultPattern` | `JOSYN.Foundation.ResultPattern` | josyn-foundation |
| `JOSYN.Foundation.PropertyBag` | `JOSYN.Foundation.PropertyBag` | josyn-foundation |
| `JOSYN.Foundation.JIP` | `JOSYN.Foundation.JIP` / `JOSYN.Foundation.JIP.Jip` | josyn-foundation |
| `JOSYN.Jap.Contract` | `JOSYN.Jap.Contract` | josyn-jap |
| `JOSYN.Jrp.Launch` / `JOSYN.Jrp.Surface` | `JOSYN.Jrp.*` | josyn-jrp (ADR-033) |
| `JOSYN.SessionBroker` | `JOSYN.SessionBroker` | josyn-session-broker (ADR-025) |
| `JOSYN.JobHost` | `JOSYN.JobHost` | josyn-job-host |
| `JOSYN.Backend.SessionStarter` | `JOSYN.Backend.SessionStarter` | josyn-backend |
| *(TBD)* | `JOSYN.Commons.*` | josyn-commons |

---

## Directory Structure Convention

> **Repo-level structure** (Pattern A vs Pattern B — single vs multi-solution repo) is defined
> in [`repo-structure-conventions.md`](repo-structure-conventions.md). The layout below
> describes how projects are arranged *within* a solution root, regardless of pattern.

Each repo organises its projects as:

```
<repo-root>/
├── <AssemblyName>/               ← directory name matches assembly name
│   ├── <AssemblyName>.csproj
│   ├── Contracts/                ← interfaces with static abstract members
│   ├── Attributes/               ← marker attributes (where applicable)
│   └── ...
├── <AssemblyName>.Test/          ← test project; directory matches assembly name
│   └── <AssemblyName>.Test.csproj
├── <AssemblyName>.slnx           ← solution file at repo root
├── nuget.config                  ← local package feed
└── .local-build/                 ← local build scripts
    ├── build.cmd
    ├── build.release.cmd
    ├── build.debug.cmd
    ├── test.cmd
    └── pack.cmd
```

---

## Contracts Folder Convention

All public static types get a companion interface with `static abstract` members in a `Contracts/` folder. This serves as API documentation and shape contract.

```
Contracts/
├── IMyStaticClass.cs    ← static abstract members match the static class
└── ...
```

Implementations reference their contract via `/// <inheritdoc cref="IXxx.Member"/>` (static classes) or `/// <inheritdoc/>` (non-static classes that formally implement the interface).

---

## File Naming

- C# files: PascalCase, one type per file, filename = type name
- Attributes: `<Name>Attribute.cs`
- Interfaces: `I<Name>.cs`

---

## Language Conventions

| Topic | Convention |
|-------|-----------|
| Error messages | **German** (`de-DE`) |
| XML documentation | **English** |
| Session/planning files | **English** |
| Culture | `de-DE` default thread culture |
