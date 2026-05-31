# Sanity State — josyn-jap

**Last checked:** 2026-05-31T12:37 UTC
**Profile:** quick (`architecture` · `standards`)

---

## architecture — ✅ Clean

### Dependency chain

| Project | References | Verdict |
|---------|-----------|---------|
| `JOSYN.Jap.Shared.Contract` | `JOSYN.Foundation.ResultPattern` only | ✅ |
| `JOSYN.Jap.Shared.Log` | `JOSYN.Foundation.ResultPattern` only | ✅ |
| `JOSYN.Jap.JAPServer` | `JOSYN.Foundation.JIP`, `JOSYN.Foundation.PropertyBag`, `JOSYN.Foundation.ResultPattern`, `JOSYN.Jap.Shared.Contract`, `JOSYN.Jap.Shared.Log` | ✅ |

No forbidden edges. No reference to `josyn-job-host` packages. No cross-contamination.

### Namespace / Assembly integrity

| Assembly | Namespace root | Verdict |
|---------|---------------|---------|
| `JOSYN.Jap.Shared.Contract` | `JOSYN.Jap.Shared.Contract` | ✅ |
| `JOSYN.Jap.Shared.Log` | `JOSYN.Jap.Shared.Log` | ✅ |
| `JOSYN.Jap.JAPServer` | `JOSYN.Jap.JAPServer` | ✅ |

### Structural / Runtime coupling

- Pipe name construction delegated entirely to `JOSYN.Foundation.JIP` (`PipesProtocol`). No hardcoded pipe name strings in josyn-jap source. ✅
- Session key parsed via `PipesProtocol.ParseSessionKeyCLIArguments(args)` — GUID-only coupling, no other shared state. ✅

---

## standards — ✅ Clean

### Project file properties

All projects satisfy universal requirements (`net10.0`, `Nullable=enable`, `AnyCPU`, `AppendTargetFrameworkToOutputPath=false`, `IncludeSourceRevisionInInformationalVersion=false`). ✅

NuGet library projects (`Shared.Contract`, `Shared.Log`): all required metadata present (`PackageId`, `Version`, `PackageLicenseExpression=MIT`, `Copyright`, `Company`, `Authors`, `PackageReadmeFile`, `PackageIcon`, `GenerateDocumentationFile=true`). `icon.png` and `README.md` co-located. ✅

EXE project (`JAPServer`): `GenerateDocumentationFile` absent, no NuGet packaging metadata. `icon.ico` present. ✅

Test project (`Shared.Log.Test`): NuGet metadata absent. Required packages `Microsoft.NET.Test.Sdk`, `NUnit 4.5.0`, `NUnit3TestAdapter` present. ✅

### Directory structure

✅ `josyn-jap-shared/` — assembly subdirectories `JOSYN.Jap.Shared.Contract/`, `JOSYN.Jap.Shared.Log/`, `JOSYN.Jap.Shared.Log.Test/` all match their assembly names exactly.

✅ `josyn-jap-japserver/JOSYN.Jap.JAPServer/` — `.csproj` and source files are inside the `JOSYN.Jap.JAPServer/` subdirectory, matching the assembly name exactly.

### `.local-build/` scripts

✅ `josyn-jap-shared/.local-build/` — `build.cmd`, `build.release.cmd`, `build.debug.cmd`, `test.cmd`, `pack.cmd`, `clean.cmd` all present.

✅ `josyn-jap-japserver/.local-build/` — `build.cmd`, `build.release.cmd`, `build.debug.cmd`, `test.cmd`, `pack.cmd` present. `launch.debug.cmd` and `launch.release.cmd` present as extras (EXE-specific convenience scripts).

### NuGet feed

Both `nuget.config` files correctly point to `..\..\local-packages\`. ✅

### `Contracts/` folders

- `JOSYN.Jap.Shared.Log` has `Contracts/ILocalLog.cs` for the public static `LocalLog` class. ✅
- `JOSYN.Jap.Shared.Contract` defines interfaces directly (no static classes requiring a companion `Contracts/` folder). ✅
- `JOSYN.Jap.JAPServer` is an EXE; `JAPServer` is a `sealed class` with instance state. No `Contracts/` required. ✅
