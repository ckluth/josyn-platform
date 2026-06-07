# ADR-012 — Maintainer Deployment (First Iteration)

**Date:** 2026-06-06
**Status:** Proposal — under construction; serves as basis for later refinement

---

## Context

The JOSYN platform is in the PoC phase. There is no CI/CD system, no installer, and no
automated staging pipeline yet. Nevertheless it is necessary to deploy the coordinated set
of artefacts (backend EXEs, adapter DLLs, job EXEs, bootstrap.ini) reproducibly and without
manual steps on a developer machine — both for integration testing and for manual
demonstration.

This ADR describes the first, deliberately simple iteration: a single PowerShell script for
maintainer use on the same machine where the local repos reside.

---

## Decision

### Deployment Script

A PowerShell script `deploy-maintainer.ps1` lives at:

```
josyn-toolbox\deploy\deploy-maintainer.ps1
```

The script is placed in `josyn-toolbox` because it:
- is not platform infrastructure (no protocol, no CI, no service)
- is maintainer tooling — deliberately outside the produced artefacts
- has dependencies on multiple repos (`josyn-backend`, `josyn-contoso`) and therefore
  does not belong cleanly in any single repo

The script uses **`dotnet publish --no-self-contained`** instead of `dotnet build + Copy-Item`.
This publishes all runtime dependencies completely and without manual selection into the
target folder.

### Target Structure

```
$BackendRoot\                            (default: C:\ProgramData\JOSYN)
    josyn.bootstrap.ini                  <- shared bootstrap configuration
    CLI\
        JOSYN.Backend.CLI.exe            <- backend CLI executable
    JAPServer\
        JOSYN.Jap.JAPServer.exe          <- JAP server executable
        Adapters\
            Contoso.Josyn.Adapter.dll    <- adapter assembly (ADR-009)

$JobRepositoryRoot\                      (default: C:\ProgramData\JOSYN\JobRepository)
    Contoso.DemoProduct.DemoJob\
        Contoso.DemoProduct.DemoJob.exe  <- first demo job
```

### bootstrap.ini Convention

`josyn.bootstrap.ini` lives at the `$BackendRoot` level — one level **above** the directory
of each trigger executable. Every trigger exe (CLI, and in future: listener, ticker, …) loads
it via:

```csharp
Path.Combine(AppContext.BaseDirectory, "..", "josyn.bootstrap.ini")
```

**Rationale:** The ini is not CLI-specific configuration — it is an installation-wide bootstrap
resource. Placing it once at `$BackendRoot` level avoids duplication and ensures that all
trigger executables see the same configuration.

### BackendRoot Convention

The directory of the loaded `josyn.bootstrap.ini` **is** the BackendRoot.
All further deployment paths are derived from it by convention —
no path is stored explicitly in the ini:

| Resource | Convention path |
|----------|----------------|
| JAPServer executable | `BackendRoot\JAPServer\JOSYN.Jap.JAPServer.exe` |
| Job repository | `BackendRoot\JobRepository\{JobTypeName}\{JobTypeName}.exe` |
| Adapters | `BackendRoot\JAPServer\Adapters\` |

`BackendRoot` itself is computed in `FileBootstrapConfig` as `Path.GetDirectoryName(iniPath)`
and exposed via `IBootstrapConfig.BackendRoot`.

### JAPServer Path Convention

The path to `JOSYN.Jap.JAPServer.exe` is **not** configured in `josyn.bootstrap.ini`.
Instead, `SessionStarter` computes it at runtime by convention:

```
<directory of the invoking trigger executable>/../JAPServer/JOSYN.Jap.JAPServer.exe
```

**Rationale:**
`SessionStarter` is the only place that needs to know `JAPServer.exe`. Beyond the current
CLI executable, further trigger executables will be introduced in future (scheduler service,
workflow adapter). All are deployed as sibling sub-folders next to `JAPServer\`. The
convention keeps this coupling consistent without any configuration overhead — at the cost
of a fixed directory structure, which this ADR makes binding.

**Consequence for deployment:**
The `JapServerExePath` key is removed from `josyn.bootstrap.ini` entirely.
`deploy-maintainer.ps1` no longer writes it.

### bootstrap.ini Adjustment

`josyn.bootstrap.ini` from the `josyn-backend` repo root is copied as-is. No keys need
to be rewritten:

| Key | Repo value | Deployment value |
|-----|-----------|-----------------|
| — | — | — |

`SessionStoreConnectionString` and `ConfigSourceType` are carried over unchanged.
All path keys are gone — they are computed by convention from BackendRoot (see above).

### Execution Order

1. Clean target folder (full delete + recreate)
2. Publish `JOSYN.Jap.JAPServer` → `$BackendRoot\JAPServer\`
3. Publish `JOSYN.Backend.CLI` → `$BackendRoot\CLI\`
4. Publish `Contoso.Josyn.Adapter` → `$BackendRoot\JAPServer\Adapters\`
5. Publish `Contoso.DemoProduct.DemoJob` → `$JobRepositoryRoot\Contoso.DemoProduct.DemoJob\`
6. Copy and adjust `josyn.bootstrap.ini` (`JobRepositoryRoot`) → `$BackendRoot\`

---

## Rationale

**Why `josyn-toolbox\deploy\`?**
The script is maintainer tooling. It has no property of a platform artefact (no protocol,
no NuGet package, no assembly). `josyn-toolbox` is the designated location for such tooling.
The toolbox constraint does not apply in this direction: the script *consumes* platform repos
— it is not referenced by them.

**Why a full delete instead of incremental update?**
In the PoC phase the target directory is not a live service folder — a clean state is more
important than minimal downtime. The maintainer deploys manually; a brief interruption is
acceptable.

**Why `--no-self-contained`?**
The developer machine has .NET 10 installed. Self-contained would significantly increase the
deployment size with no benefit for this iteration.

---

## Open Questions (Refinement Basis)

These questions are explicitly open — this ADR does not close them, but documents them as
input for a future deployment architecture:

1. **Staging environments:** How is the distinction between DEV, INT, and PROD handled?
   Separate bootstrap.ini per environment? Environment parameter in the script?
   (See ADR-010)

2. **Job repository convention:** The `JobRepositoryRoot` concept is modelled in
   `IBootstrapConfig` as a simple root path. If jobs require multiple files (configuration,
   icons, manifest), each job sub-folder needs a defined structure. That structure has not
   been decided yet.

3. **Adapter loading convention:** The `Adapters\` sub-directory next to the backend EXE is
   a convention from ADR-009, physically implemented here for the first time. Whether multiple
   adapters can coexist side by side has not been specified.

4. **CI/CD:** This script is explicitly not a CI build step. If a pipeline is introduced, the
   deployment model must be redesigned (staging slots, artifact upload, rollback strategy).

5. **Installer vs. script:** For a real production environment a Windows installer (MSI/WiX)
   or a packaging format is the natural next step. That is outside the current scope.

6. **Multiple jobs:** Currently `Contoso.DemoProduct.DemoJob` is the only job. When further
   jobs are added the script must be extended or parameterised (e.g., a loop over a list of
   job solutions).

---

## Consequences

- The maintainer can create a complete, consistent deployment environment on the local machine
  with a single `pwsh .\deploy-maintainer.ps1` call.
- The exact target structure is documented and reproducible.
- Open questions are explicitly recorded — future refinement can pick up directly from here.

---

## Relation to Other ADRs

- **ADR-009** (Runtime Context Provider / Adapter Pattern): The `Adapters\` directory is the
  physical realisation of the adapter loading mechanism described there.
- **ADR-010** (Environment Separation): This ADR addresses DEV only.
  INT and PROD are explicitly open questions (see above).
