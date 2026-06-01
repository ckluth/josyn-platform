# Sanity State — josyn-jap

**Last checked:** 2026-05-31T12:37 UTC
**Profile:** quick (`architecture` · `standards`)

> **Note:** `JOSYN.Jap.JAPServer` was relocated to `josyn-backend` per ADR-004 (2026-06-01).
> Its architecture and standards checks are now part of the josyn-backend state file.
> The checks below cover the two remaining packages only.

---

## architecture — ✅ Clean

### Dependency chain

| Project | References | Verdict |
|---------|-----------|---------|
| `JOSYN.Jap.Shared.Contract` | `JOSYN.Foundation.ResultPattern` only | ✅ |
| `JOSYN.Jap.Shared.Log` | `JOSYN.Foundation.ResultPattern` only | ✅ |

No forbidden edges. No reference to `josyn-job-host` packages. No EXE project in repo.

### Namespace / Assembly integrity

| Assembly | Namespace root | Verdict |
|---------|---------------|---------|
| `JOSYN.Jap.Shared.Contract` | `JOSYN.Jap.Shared.Contract` | ✅ |
| `JOSYN.Jap.Shared.Log` | `JOSYN.Jap.Shared.Log` | ✅ |

### Structural / Runtime coupling

- Shared packages have no runtime spawn responsibilities — contracts only. ✅
- No hardcoded pipe names in either package. ✅

---

## standards — ✅ Clean

### Project file properties

All projects satisfy universal requirements (`net10.0`, `Nullable=enable`, `AnyCPU`, `AppendTargetFrameworkToOutputPath=false`, `IncludeSourceRevisionInInformationalVersion=false`). ✅

NuGet library projects (`Shared.Contract`, `Shared.Log`): all required metadata present (`PackageId`, `Version`, `PackageLicenseExpression=MIT`, `Copyright`, `Company`, `Authors`, `PackageReadmeFile`, `PackageIcon`, `GenerateDocumentationFile=true`). `icon.png` and `README.md` co-located. ✅

Test project (`Shared.Log.Test`): NuGet metadata absent. Required packages `Microsoft.NET.Test.Sdk`, `NUnit 4.5.0`, `NUnit3TestAdapter` present. ✅

### Directory structure

✅ `josyn-jap-shared/` — assembly subdirectories `JOSYN.Jap.Shared.Contract/`, `JOSYN.Jap.Shared.Log/`, `JOSYN.Jap.Shared.Log.Test/` all match their assembly names exactly.

### `.local-build/` scripts

✅ `josyn-jap-shared/.local-build/` — `build.cmd`, `build.release.cmd`, `build.debug.cmd`, `test.cmd`, `pack.cmd`, `clean.cmd` all present.

### NuGet feed

`nuget.config` correctly points to `..\..\local-packages\`. ✅

### `Contracts/` folders

- `JOSYN.Jap.Shared.Log` has `Contracts/ILocalLog.cs` for the public static `LocalLog` class. ✅
- `JOSYN.Jap.Shared.Contract` defines interfaces directly (no static classes requiring a companion `Contracts/` folder). ✅
