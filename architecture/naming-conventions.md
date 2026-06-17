# Naming Conventions

## Vocabulary Map

| Word | Scope | Usage |
|------|-------|-------|
| **Platform** | The entire JOSYN ecosystem | Umbrella term — all repos, all layers |
| **Backend** | The scheduler and session-orchestration layer | `josyn-backend` repo, `JOSYN.Backend.*` namespace |
| **JAP** | The per-session JAP protocol server and shared packages | `josyn-jap` repo, `JOSYN.Jap.*` namespace |
| **Foundation** | Cross-cutting infrastructure primitives | `josyn-foundation` repo, `JOSYN.Foundation.*` namespace |
| **Job Host** | The job execution runtime | `josyn-job-host` repo, `JOSYN.JobHost` namespace |
| **Commons** | Generic utility helpers — domain-agnostic, open for growth | `josyn-commons` repo, `JOSYN.Commons.*` namespace |
| **Adapter** | The boundary layer between the platform and company adapter EXEs | `josyn-adapter-contracts` repo, `JOSYN.Adapter.*` namespace |

> **Note:** "JAP" does **not** mean "backend" in this codebase. `josyn-jap` is the per-session JAP server, not the scheduler. The scheduler role belongs exclusively to `josyn-backend`. See [../decisions/ADR-002-josyn-backend.md](../decisions/ADR-002-josyn-backend.md).

> **Decision:** "Platform" was chosen as the umbrella word because "JAP" already carries a specific architectural meaning (the JAP session server). See [../decisions/ADR-001-platform-naming.md](../decisions/ADR-001-platform-naming.md).

---

## Repository Names

```
josyn-foundation        ← infrastructure primitives
josyn-jap               ← per-session JAP protocol server
josyn-job-host          ← job execution runtime  
josyn-backend           ← scheduler and session-orchestration layer
josyn-commons           ← generic utility helpers — domain-agnostic, open for growth
josyn-adapter-contracts ← JIP protocol contracts for company adapter EXEs (ADR-023)
josyn-platform          ← this repo; architecture + decisions + docs
```

Pattern: `josyn-<layer>` — all lowercase, hyphen-separated.

The absence of "jap" in `josyn-job-host` (despite the `JOSYN.Jap.*` layer packages it depends on) is **deliberate**: it signals that job executables are architectural outsiders — decoupled consumers of the protocol, not internal components of the scheduler. This reasoning is extended to the namespace: `JOSYN.JobHost` (not `JOSYN.Jap.JobHost`) hides the internal "Jap" protocol layer from the developer who just wants to author a job.

> **Intentional naming exception — documented rule:**  
> A package whose primary audience is *external job authors* (consumers outside the JOSYN platform) **omits the `<Layer>` segment** and uses the two-segment form `JOSYN.<Component>`. `JOSYN.JobHost` is the canonical and currently only example. All platform-internal packages follow the three-segment `JOSYN.<Layer>.<Component>` pattern without exception. See [../decisions/ADR-001-platform-naming.md](../decisions/ADR-001-platform-naming.md).

---

## Namespace Conventions

```
JOSYN
├── Foundation                     ← josyn-foundation packages
│   ├── ResultPattern              ← error-as-value primitives
│   ├── PropertyBag                ← record serialization
│   └── JIP                        ← named pipe transport
│       └── Jip                    ← JIP convention layer
├── Jap                            ← josyn-jap (platform-internal)
│   ├── Shared
│   │   ├── Contract               ← IJosynApplicationProtocol, ErrorReport
│   │   └── Log                    ← LocalLog
│   └── JAPServer                  ← per-session JAP server EXE (lives in josyn-backend)
├── Adapter                        ← josyn-adapter-contracts (ADR-023)
│   ├── ConfigurationAdapter
│   │   └── Contract               ← IConfigurationAdapter
│   └── IdentityAdapter
│       └── Contract               ← IIdentityAdapter
├── JobHost                        ← josyn-job-host (consumer-facing API — see note below)
│   └── Attributes                 ← [JobEntryPoint], [BeforeJobEntryPoint], etc.
├── Backend                        ← josyn-backend
│   └── SessionStarter             ← session lifecycle rendezvous (stub)
└── Commons                        ← josyn-commons (utility satellite — never referenced by Foundation)
    └── ...                        ← packages added as helpers accumulate
```

Pattern: `JOSYN.<Layer>.<Component>[.<Subcomponent>]` — with one intentional exception:  
`JOSYN.JobHost` uses the two-segment form because it is a **consumer-facing API** (see the rule under *Repository Names* above).

---

## Project / Assembly Names

Assembly names match their namespace root exactly:

| Assembly | Namespace | Repo |
|----------|-----------|------|
| `JOSYN.Foundation.ResultPattern` | `JOSYN.Foundation.ResultPattern` | josyn-foundation |
| `JOSYN.Foundation.PropertyBag` | `JOSYN.Foundation.PropertyBag` | josyn-foundation |
| `JOSYN.Foundation.JIP` | `JOSYN.Foundation.JIP` / `JOSYN.Foundation.JIP.Jip` | josyn-foundation |
| `JOSYN.Jap.Contract` | `JOSYN.Jap.Contract` | josyn-jap |
| `JOSYN.Jap.Shared.Log` | `JOSYN.Jap.Shared.Log` | josyn-jap |
| `JOSYN.Jap.JAPServer` | `JOSYN.Jap.JAPServer` | josyn-backend (relocated from josyn-jap per ADR-004) |
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
