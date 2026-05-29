# Naming Conventions

## Vocabulary Map

| Word | Scope | Usage |
|------|-------|-------|
| **Platform** | The entire JOSYN ecosystem | Umbrella term — all repos, all layers |
| **Backend** | The scheduler and session-orchestration layer | `josyn-backend` repo, `JOSYN.Backend.*` namespace |
| **JAP** | The per-session JAP protocol server and shared packages | `josyn-jap` repo, `JOSYN.Jap.*` namespace |
| **Foundation** | Cross-cutting infrastructure primitives | `josyn-foundation` repo, `JOSYN.Foundation.*` namespace |
| **Job Host** | The job execution runtime | `josyn-job-host` repo, `JOSYN.Jap.JobHost` namespace |

> **Note:** "JAP" does **not** mean "backend" in this codebase. `josyn-jap` is the per-session JAP server, not the scheduler. The scheduler role belongs exclusively to `josyn-backend`. See [../decisions/ADR-002-josyn-backend.md](../decisions/ADR-002-josyn-backend.md).

> **Decision:** "Platform" was chosen as the umbrella word because "JAP" already carries a specific architectural meaning (the JAP session server). See [../decisions/ADR-001-platform-naming.md](../decisions/ADR-001-platform-naming.md).

---

## Repository Names

```
josyn-foundation        ← infrastructure primitives
josyn-jap               ← per-session JAP protocol server
josyn-job-host          ← job execution runtime  
josyn-backend           ← scheduler and session-orchestration layer
josyn-platform          ← this repo; architecture + decisions + docs
```

Pattern: `josyn-<layer>` — all lowercase, hyphen-separated.

The absence of "jap" in `josyn-job-host` (despite the namespace containing `JOSYN.Jap.JobHost`) is **deliberate**: it signals that job executables are architectural outsiders — decoupled consumers of the protocol, not internal components of the scheduler.

---

## Namespace Conventions

```
JOSYN
├── Foundation                     ← josyn-foundation packages
│   ├── ResultPattern              ← error-as-value primitives
│   ├── PropertyBag                ← record serialization
│   └── JIP                        ← named pipe transport
│       └── Jip                    ← JIP convention layer
├── Jap                            ← josyn-jap + josyn-job-host
│   ├── Shared
│   │   ├── Contract               ← IJosynApplicationProtocol, ErrorReport
│   │   └── Log                    ← LocalLog
│   ├── JAPServer                  ← per-session JAP server EXE
│   └── JobHost                    ← job execution runtime library
│       └── Attributes             ← [JobEntryPoint], [BeforeJobEntryPoint], etc.
└── Backend                        ← josyn-backend
    └── SessionStarter             ← session lifecycle rendezvous (stub)
```

Pattern: `JOSYN.<Layer>.<Component>[.<Subcomponent>]`

---

## Project / Assembly Names

Assembly names match their namespace root exactly:

| Assembly | Namespace | Repo |
|----------|-----------|------|
| `JOSYN.Foundation.ResultPattern` | `JOSYN.Foundation.ResultPattern` | josyn-foundation |
| `JOSYN.Foundation.PropertyBag` | `JOSYN.Foundation.PropertyBag` | josyn-foundation |
| `JOSYN.Foundation.JIP` | `JOSYN.Foundation.JIP` / `JOSYN.Foundation.JIP.Jip` | josyn-foundation |
| `JOSYN.Jap.Shared.Contract` | `JOSYN.Jap.Shared.Contract` | josyn-jap |
| `JOSYN.Jap.Shared.Log` | `JOSYN.Jap.Shared.Log` | josyn-jap |
| `JOSYN.Jap.JAPServer` | `JOSYN.Jap.JAPServer` | josyn-jap |
| `JOSYN.Jap.JobHost` | `JOSYN.Jap.JobHost` | josyn-job-host |
| `JOSYN.Backend.SessionStarter` | `JOSYN.Backend.SessionStarter` | josyn-backend |

---

## Directory Structure Convention

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
├── Directory.Build.props         ← shared build output paths
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
- Generic variants: `<Name>.generic.cs` (e.g. `Result.generic.cs`)

---

## Language Conventions

| Topic | Convention |
|-------|-----------|
| Error messages | **German** (`de-DE`) |
| XML documentation | **English** |
| Session/planning files | **English** |
| Culture | `de-DE` default thread culture |
