# Sanity State — josyn-foundation

**Last checked:** 2026-05-31T11:41 UTC

---

## Summary

| Category | Status |
|----------|--------|
| `docs` | ✅ Pass |
| `tests` | ✅ Pass |
| `principles` | ❌ Violations found |
| `architecture` | ❌ Violation found |
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

## principles ❌

### P1 — Static wins

**5 violations**: The following types have zero instance state and no polymorphism need, but are declared as `sealed class` rather than `static class`. The established pattern in this repo (see `JipClient`, `JipServer`, `IniDictionarySerializer`, `JsonDictionarySerializer`) is `static class` with `<inheritdoc cref="IXxx.Member"/>` doc comments — no formal interface implementation at the syntax level.

| Type | File | Signal |
|------|------|--------|
| `PropertyBag` | `josyn-foundation-property-bag/JOSYN.Foundation.PropertyBag/PropertyBag.cs` | `sealed class` with zero instance state; all members `public static` |
| `PipesClient` | `josyn-foundation-jip/JOSYN.Foundation.JIP/PipesClient.cs` | `sealed class` with zero instance state; all members `public static` |
| `PipesServer` | `josyn-foundation-jip/JOSYN.Foundation.JIP/PipesServer.cs` | `sealed class` with zero instance state; all members `public static` (private inner classes `RawRequestHandler` and `CancellationHandle` have state, but these are helpers, not state of `PipesServer` itself) |
| `PipesProtocol` | `josyn-foundation-jip/JOSYN.Foundation.JIP/PipesProtocol.cs` | `sealed class` with zero instance state; all members `public static` |
| `JipProtocol` | `josyn-foundation-jip/JOSYN.Foundation.JIP/Jip/JipProtocol.cs` | `sealed class` with zero instance state; all members `public static` |

**Recommended fix:** Convert each to `static class`, drop the `: IXxx` formal interface implementation, and switch docs to `/// <inheritdoc cref="IXxx.Member"/>` (see `JipClient.cs` as the reference implementation).

### P3 — Errors as values, never exceptions

**3 violations**: `throw` above the lowest-layer catch boundary.

| Location | Code | Note |
|----------|------|------|
| `Result.cs` line 101 | `throw new InvalidOperationException("ToResult<T>() wurde auf einem succeeded Result aufgerufen.")` | Called from `ToResult<TValue>()` — a misuse guard. Could return `Result<TValue>.Fail(...)` instead. |
| `Result.generic.cs` line 109 | `throw new InvalidOperationException("ToResult<T>() wurde auf einem succeeded Result aufgerufen.")` | Same pattern in `Result<T>.ToResult<TOther>()`. |
| `JipDispatcher.cs` line 89 | `throw new InvalidOperationException($"Methode '{typeof(TProtocol).Name}.{key}' hat eine nicht unterstützte Signatur...")` | In `RegisterAll` — a programming-time validation. Method signature would need to return `Result<IJipDispatcher>` to avoid the throw. |

**Context for items 1 & 2:** The `ToResult<T>()` overloads are explicitly documented as "call only on a failed result". The tests confirm and test the throw behavior. The intent is a programming-time guard, analogous to `Span<T>` misuse guards. Whether this warrants a signature change (returning `Result<T>` from an always-failed path) is a design decision. Currently a violation by strict reading of the principle.

**Context for item 3:** `RegisterAll` is called at setup time, not in the hot path. A failure here always represents a programming error (wrong method signature on the protocol interface). Same tradeoff as above.

### P5 — Explicit over magic

**1 finding** (lower confidence):

`JipDispatcher.RegisterAll<TProtocol>` uses reflection to enumerate methods on `TProtocol` and auto-register them as handlers by method name. This is reflection-based wiring. The exception in the criteria is specifically the `[JobEntryPoint]` dispatch — `RegisterAll` is not that mechanism.

**Mitigating factors:** The method is explicitly called, explicitly documented in `IJipDispatcher`, and the auto-wiring is opt-in per call. It is not hidden convention magic. The risk is marginally higher than a direct `Register("key", handler)` call but the explicitness of the call site partially satisfies the principle. Flag as a finding; resolve by deciding whether explicit manual `Register(...)` calls are preferred over `RegisterAll`.

---

## architecture ❌

### Pipe naming deviation

**1 violation:** `PipesProtocol.DerivePipeNamesFromSessionKey` produces pipe names `"req-pipe-{sessionKey}"` and `"res-pipe-{sessionKey}"`. The architecture overview (`architecture/overview.md`) documents the names as `JOSYN-REQ-<sessionGUID>` and `JOSYN-RSP-<sessionGUID>`.

| Check | Status |
|-------|--------|
| Pipe names deviate from `JOSYN-REQ-<GUID>` / `JOSYN-RSP-<GUID>` pattern | ❌ violation |

**File:** `josyn-foundation-jip/JOSYN.Foundation.JIP/PipesProtocol.cs`

**Context:** The `IPipesProtocol` contract documentation and tests are consistent with the `req-pipe-` / `res-pipe-` format. The architecture overview is out of sync with the implementation. One of the two must be corrected:
- Option A: Update `architecture/overview.md` to document `req-pipe-<sessionKey>` / `res-pipe-<sessionKey>` as the actual pipe naming scheme.
- Option B: Change the implementation to use `JOSYN-REQ-<GUID>` / `JOSYN-RSP-<GUID>` and update `IPipesProtocol` docs and tests accordingly.

Option A is lower risk (no runtime impact). Option B aligns with the documented standard but requires a coordinated change across all consumers (josyn-jap, josyn-job-host).

All other architecture checks pass:
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

**Known PoC limitations (documented in `POC-HACKS.md`, not violations):**
- `Directory.Build.props` uses hardcoded `C:\Temp\VS.OUT\JOSYN\` build output path
- Local NuGet feed path — production-only concern

