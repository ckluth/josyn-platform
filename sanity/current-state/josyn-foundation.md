# Sanity State — josyn-foundation

**Last checked:** 2026-05-31T11:41 UTC (violations fixed at 2026-05-31T12:10 UTC)

---

## Summary

| Category | Status |
|----------|--------|
| `docs` | ✅ Pass |
| `tests` | ✅ Pass |
| `principles` | ✅ Pass (fixed) |
| `architecture` | ✅ Pass (fixed) |
| `standards` | ✅ Pass |

---

## docs ✅

All criteria pass.

- All public interfaces carry complete XML documentation in English.
- Implementations use `<inheritdoc/>` (non-static) or `<inheritdoc cref="..."/>` (static) — never duplicate contract text.
- No `<exception>` tags anywhere.
- `GenerateDocumentationFile=true` present in all three NuGet library `.csproj` files; absent from all three test `.csproj` files.
- Each sub-package has a `README.md` co-located with its `.csproj`.
- Root `README.md` exists and accurately describes the three packages, their dependency chain, and build workflow.
- **Known exception (documented):** `JOSYN.Foundation.ResultPattern` keeps its `README.md` one level above the `.csproj` — legacy layout, intentional per `repos/josyn-foundation.md`.

---

## tests ✅

All criteria pass.

- **141 tests** pass for `JOSYN.Foundation.ResultPattern.Test` — zero failures, zero skips.
- **61 tests** pass for `JOSYN.Foundation.PropertyBag.Test` — zero failures, zero skips.
- **48 tests** pass for `JOSYN.Foundation.JIP.Test` — zero failures, zero skips.
- **Total: 250 tests, 0 failures.**
- Test naming consistently follows `SubjectUnderTest_Scenario_ExpectedResult`.
- Fluent assertions used throughout (`Assert.That(x, Is.EqualTo(y))`).
- No `[Ignore]` attributes, no `[ExpectedException]` attributes.
- Arrange-Act-Assert structure observed; no flow control (loops/conditionals) inside test bodies.
- `Assert.Throws<TException>(() => ...)` used correctly for the two documented `throw` cases.
- Test helpers (static factory methods, private scenario helpers) used appropriately.

**Coverage note:** No integration tests cover the actual named pipe transport (`PipesClient` / `PipesServer` end-to-end). This is acceptable for PoC stage. Not a violation, but worth tracking.

---

## principles ✅

All violations resolved.

- **P1 (5 fixes):** `PropertyBag`, `PipesClient`, `PipesServer`, `PipesProtocol`, and `JipProtocol` converted from `sealed class : IXxx` to `static class`. Formal interface implementation dropped (static classes cannot implement interfaces). All `/// <inheritdoc/>` comments updated to explicit `/// <inheritdoc cref="IXxx.Member"/>` form.
- **P3 (3 fixes):**
  - `Result.ToResult<TValue>()` on a succeeded result now returns `Result<TValue>.Fail(...)` instead of throwing. Test updated from `Throws` assertion to failure-result assertion.
  - `Result<T>.ToResult<TOther>()` on a succeeded result — same fix. Test updated.
  - `JipDispatcher.RegisterAll<TProtocol>()` return type changed from `IJipDispatcher` to `Result<IJipDispatcher>`. On unsupported signature: returns `Result<IJipDispatcher>.Fail(...)` instead of throwing. Interface updated. All callers in josyn-foundation updated (tests + demo `Program.cs`).
- **P5:** Left as-is — `RegisterAll` reflection is explicitly called and opt-in; not hidden convention magic. Confirmed design decision.

**Tests:** 251 total (141 + 61 + 49), 0 failures after all fixes.

---

## architecture ✅

### Pipe naming — resolved

`architecture/overview.md` updated to document the actual pipe naming scheme (`req-pipe-<sessionKey>` / `res-pipe-<sessionKey>`), matching `PipesProtocol.DerivePipeNamesFromSessionKey`, `IPipesProtocol` contract docs, and existing tests.

All other architecture checks continue to pass:
- Zero outbound NuGet dependencies from `JOSYN.Foundation.ResultPattern` ✅
- `JOSYN.Foundation.PropertyBag` → `ResultPattern` only ✅
- `JOSYN.Foundation.JIP` → `ResultPattern` only ✅
- No forbidden cross-repo dependency edges ✅
- Assembly names match namespace roots for all three packages ✅
- Session GUID isolation pattern confirmed (`MagicToken = "JOSYN-IPC"`, GUID passed as CLI argument) ✅

---

## standards ✅

All criteria pass.

**Project file properties (all three NuGet library projects):**

| Property | Value | Status |
|----------|-------|--------|
| `<TargetFramework>` | `net10.0` | ✅ |
| `<Nullable>` | `enable` | ✅ |
| `<PlatformTarget>` | `AnyCPU` | ✅ |
| `<AppendTargetFrameworkToOutputPath>` | `false` | ✅ |
| `<IncludeSourceRevisionInInformationalVersion>` | `false` | ✅ |
| `<OutputType>` | `Library` | ✅ |
| `<GenerateDocumentationFile>` | `true` | ✅ |
| `<PackageLicenseExpression>` | `MIT` | ✅ |
| `<Copyright>` | `Copyright © 2026 HAEVG AG` | ✅ |
| `<Company>` | `HAEVG` | ✅ |
| `<Authors>` | `HAEVG SWE` | ✅ |
| `<Version>` | `1.0.0-preview01` | ✅ |
| `<PackageId>` | Matches `<AssemblyName>` | ✅ |
| `<PackageReadmeFile>` | `README.md` | ✅ |
| `<PackageIcon>` | `icon.png` | ✅ |
| `icon.png` | Co-located with `.csproj` | ✅ |

**Test project properties:** `GenerateDocumentationFile` absent, NuGet metadata absent, `Microsoft.NET.Test.Sdk` + `NUnit 4.x` + `NUnit3TestAdapter` present in all three test projects ✅

**Directory structure:** All three sub-packages follow `<AssemblyName>/` + `<AssemblyName>.Test/` + `<AssemblyName>.slnx` layout ✅

**nuget.config:** All three point to `..\..\local-packages\` ✅

**`.local-build/` scripts:** All required scripts (`build.cmd`, `build.release.cmd`, `build.debug.cmd`, `test.cmd`, `pack.cmd`) present in all three sub-packages ✅

**Naming conventions:**
- Repo follows `josyn-<layer>` pattern ✅
- Assembly names match namespace roots (`JOSYN.Foundation.ResultPattern`, `JOSYN.Foundation.PropertyBag`, `JOSYN.Foundation.JIP`) ✅
- `Contracts/` folders present in all three source projects ✅
- File names match type names throughout ✅
- Interface files prefixed with `I` ✅

**Known PoC limitations:**
- Local NuGet feed path — production-only concern

