# Sanity Criteria — standards

> Verify naming conventions, project file structure, and directory layout.
> Read `architecture/naming-conventions.md` before evaluating.

---

## Project File Standards

### Universal rules (all project types)

| Property | Required value |
|----------|---------------|
| `<TargetFramework>` | `net10.0` |
| `<Nullable>` | `enable` |
| `<PlatformTarget>` | `AnyCPU` |
| `<AppendTargetFrameworkToOutputPath>` | `false` |
| `<IncludeSourceRevisionInInformationalVersion>` | `false` |

### NuGet library projects (Template 1)

Additional required properties:

| Property | Required value |
|----------|---------------|
| `<OutputType>` | `Library` |
| `<GenerateDocumentationFile>` | `true` |
| `<PackageLicenseExpression>` | `MIT` |
| `<Copyright>` | `Copyright © 2026 HAEVG AG` |
| `<Company>` | `HAEVG` |
| `<Authors>` | `HAEVG SWE` |
| `<Version>` | Present — a concrete version number |
| `<PackageId>` | Matches `<AssemblyName>` exactly |
| `<PackageReadmeFile>` | `README.md` (declared once) |
| `<PackageIcon>` | `icon.png` |
| `icon.png` | Co-located with the `.csproj` |
| `README.md` | Co-located with the `.csproj` *(exception: ResultPattern keeps README one level up — legacy layout)* |

### Exe / non-NuGet projects (Template 2)

| Property | Rule |
|----------|------|
| `<GenerateDocumentationFile>` | **Absent** — must not be present |
| NuGet metadata (`PackageId`, `Description`, etc.) | **Absent** |
| `icon.ico` | Co-located with the `.csproj` (Windows Explorer icon) |

### Test projects (Template 3)

| Property | Rule |
|----------|------|
| `<GenerateDocumentationFile>` | **Absent** |
| NuGet metadata | **Absent** |
| Required package refs | `Microsoft.NET.Test.Sdk`, `NUnit 4.x`, `NUnit3TestAdapter` |

---

## Directory Structure

Each repo must follow:

```
<repo-root>/
├── <AssemblyName>/               ← directory name matches assembly name exactly
│   ├── <AssemblyName>.csproj
│   ├── Contracts/                ← companion interfaces (static abstract members)
│   └── ...
├── <AssemblyName>.Test/          ← test project directory matches assembly name
│   └── <AssemblyName>.Test.csproj
├── <AssemblyName>.slnx           ← solution file at repo root
├── Directory.Build.props         ← shared build output redirect
├── nuget.config                  ← local package feed at `..\..\local-packages\`
└── .local-build/
    ├── build.cmd
    ├── build.release.cmd
    ├── build.debug.cmd
    ├── test.cmd
    └── pack.cmd
```

| Check | Verdict |
|-------|---------|
| Directory name differs from assembly name | ❌ violation |
| `Contracts/` folder absent from a public static type's project | ❌ violation |
| Solution file not at repo root | ❌ violation |
| `nuget.config` missing or not pointing to `..\..\local-packages\` | ❌ violation |
| `.local-build/` scripts missing | ❌ violation |

---

## Naming Conventions

See `architecture/naming-conventions.md` for the full vocabulary map, namespace tree, and assembly table.

| Check | Verdict |
|-------|---------|
| Repo name does not follow `josyn-<layer>` pattern | ❌ violation |
| Namespace does not follow `JOSYN.<Layer>.<Component>` (except `JOSYN.JobHost` — intentional) | ❌ violation |
| Assembly name differs from namespace root | ❌ violation |
| C# file name differs from type name | ❌ violation |
| Interface file not prefixed with `I` | ❌ violation |

---

## NuGet Feed

- Each repo's `nuget.config` must declare the local feed at `..\..\local-packages\` (relative to repo root).
- All platform packages are resolved from this local feed.
- No public NuGet feed references for platform packages.
