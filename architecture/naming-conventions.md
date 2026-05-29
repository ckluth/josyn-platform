# Naming Conventions

## Vocabulary Map

| Word | Scope | Usage |
|------|-------|-------|
| **Platform** | The entire JOSYN ecosystem | Umbrella term — all repos, all layers |
| **System** | The orchestration/backend layer | `josyn-system` repo, `JOSYN.System.*` namespace |
| **Foundation** | Cross-cutting infrastructure primitives | `josyn-foundation` repo, `JOSYN.Foundation.*` namespace |
| **Job Host** | The job execution runtime | `josyn-job-host` repo, `JOSYN.System.JobHost` namespace |

> **Decision:** "Platform" was chosen as the umbrella word because "system" already carries a specific architectural meaning in this codebase (the orchestration backend). See [../decisions/ADR-001-platform-naming.md](../decisions/ADR-001-platform-naming.md).

---

## Repository Names

```
josyn-foundation        ← infrastructure primitives
josyn-system            ← orchestration backend
josyn-job-host          ← job execution runtime  
josyn-platform          ← this repo; architecture + decisions + docs
```

Pattern: `josyn-<layer>` — all lowercase, hyphen-separated.

The absence of "system" in `josyn-job-host` (despite the namespace containing `JOSYN.System.JobHost`) is **deliberate**: it signals that job executables are architectural outsiders — decoupled consumers of the protocol, not internal components of the scheduler.

---

## Namespace Conventions

```
JOSYN                              ← root org
├── Foundation                     ← josyn-foundation packages
│   ├── ResultPattern              ← error-as-value primitives
│   ├── PropertyBag                ← record serialization
│   └── JIP                        ← named pipe transport
│       └── Jip                    ← JIP convention layer
└── System                         ← josyn-system + josyn-job-host
    ├── Shared
    │   ├── Contract               ← IJosynApplicationProtocol, ErrorReport
    │   └── Log                    ← LocalLog
    ├── JAPServer                  ← backend server EXE
    └── JobHost                    ← job execution runtime library
        └── Attributes             ← [JobEntryPoint], [BeforeJobEntryPoint], etc.
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
| `JOSYN.System.Shared.Contract` | `JOSYN.System.Shared.Contract` | josyn-system |
| `JOSYN.System.Shared.Log` | `JOSYN.System.Shared.Log` | josyn-system |
| `JOSYN.System.JAPServer` | `JOSYN.System.JAPServer` | josyn-system |
| `JOSYN.System.JobHost` | `JOSYN.System.JobHost` | josyn-job-host |

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
