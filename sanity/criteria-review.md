# Sanity Criteria — Decision-Maker Review

> **Living document.** Revisit periodically — especially after adding new criteria, after a full
> sanity run uncovers surprising noise, or when the platform architecture evolves.
>
> **Purpose:** Judge every criterion by its signal quality so the sanity check stays tight
> and high-value. Low-signal criteria erode trust in the whole check.
>
> **Workflow:** Review the 🟡 and 🔴 items. For each, decide: Accept / Weaken / Drop.
> Then reconcile the source files (`architecture.md`, `standards.md`, `docs.md`, `tests.md`, `principles.md`).
>
> **Rating legend**
>
> | Symbol | Meaning |
> |--------|---------|
> | 🟢 | Indispensable — catches real problems, no false positives, keep as-is |
> | 🟡 | Discussable — reasonable rule, but enforcement may be noisy or the value is limited |
> | 🔴 | Not really necessary — nice-to-have at best; adds noise relative to signal |
> | ⛔ | Bare mistake — actively wrong to check this, or rule itself is flawed |

---

## 1. Architecture

| # | Criterion | Rating | Rationale |
|---|-----------|--------|-----------|
| A1 | Permitted dependency edges (josyn-foundation → nothing, etc.) | 🟢 | Core architectural invariant. A wrong edge silently breaks the isolation model at runtime. |
| A2 | Forbidden edge: backend → josyn-jap NuGet packages | 🟢 | If this edge forms, the process-isolation design collapses into tight coupling. |
| A3 | Forbidden edge: foundation → anything | 🟢 | Foundation is the root of all trees; a cycle here is a structural catastrophe. |
| A4 | Forbidden edge: jap ↔ job-host (symmetric peers, never reference each other) | 🟢 | They are equal participants in the IPC protocol, not layers. Coupling them breaks the model. |
| A5 | Forbidden edge: any repo → josyn-platform | 🟢 | Platform is documentation only. A csproj reference to it would be meaningless and confusing. |
| A6 | josyn-commons must not reference platform packages beyond ResultPattern | 🟢 | Commons is a utility satellite. Extra deps defeat its purpose. |
| A7 | Backend spawns JAPServer.exe — session GUID is the only coupling | 🟢 | Verifies the IPC contract is not being bypassed by other shared state. |
| A8 | Named pipe names follow `JOSYN-REQ-<sessionGUID>` / `JOSYN-RSP-<sessionGUID>` | 🟡 | Hard to check statically (no string search is reliable). Only discoverable at runtime or via explicit code search. Consider limiting to "flag hardcoded pipe names that deviate." |
| A9 | Assembly name = namespace root (with documented `JOSYN.JobHost` exception) | 🟢 | Fundamental .NET convention; violating it breaks tooling, IDE navigation, and build output. |

---

## 2. Standards — Project File

### Universal (all project types)

| # | Criterion | Rating | Rationale |
|---|-----------|--------|-----------|
| S1 | `<TargetFramework>` = `net10.0` | 🟢 | Mixed target frameworks across the platform will cause package incompatibilities. |
| S2 | `<Nullable>enable</Nullable>` | 🟢 | Disabling nullability breaks a key safety guarantee and creates noise for partial-enable edge cases. |
| S3 | `<ImplicitUsings>enable</ImplicitUsings>` | 🟡 | Consistency matters but violation is harmless — extra explicit `using` statements are fine. |
| S4 | `<LangVersion>latest</LangVersion>` | 🟡 | Guarantees latest C# features are available, but `net10.0` already implies a minimum. "Latest" can surprise teams on tooling updates. Consider locking to a concrete version. |
| S5 | `<PlatformTarget>AnyCPU</PlatformTarget>` | 🟢 | x86-specific builds break on ARM/Linux toolchains. Enforcing AnyCPU prevents silent platform lockout. |
| S6 | `<AppendTargetFrameworkToOutputPath>false</AppendTargetFrameworkToOutputPath>` | 🟡 | Affects output path structure. Important for `.local-build/` scripts to work predictably, but a naming convention, not a correctness rule. |
| S7 | `<IncludeSourceRevisionInInformationalVersion>false</IncludeSourceRevisionInInformationalVersion>` | 🔴 | Affects version metadata only. Low signal: no one is likely to be hurt by having the git hash in informational version. |
| S8 | Indentation = tabs (not spaces) in csproj | 🔴 | Cosmetic. Editors reformat on save. A sanity check that fails over whitespace is noise. |
| S9 | Single `<PropertyGroup>` (no splits unless conditional) | 🔴 | Cosmetic preference. A second group doesn't break anything and is sometimes clearer. |

### NuGet library projects

| # | Criterion | Rating | Rationale |
|---|-----------|--------|-----------|
| S10 | `<OutputType>Library</OutputType>` | 🟢 | Absence defaults to Exe for non-test projects — would accidentally produce an executable. |
| S11 | `<GenerateDocumentationFile>true</GenerateDocumentationFile>` | 🟢 | Without this, XML warnings are silenced, documentation is missing from the package, and the `docs` check has no baseline. |
| S12 | `<PackageLicenseExpression>MIT</PackageLicenseExpression>` | 🟢 | Missing license is a compliance issue for any published package. |
| S13 | `<Copyright>`, `<Company>`, `<Authors>` | 🟡 | Metadata integrity. Matters for published packages; less critical for private internal packages. |
| S14 | `<PackageProjectUrl>` / `<RepositoryUrl>` | 🟡 | Metadata only. Useful for NuGet.org discoverability, irrelevant for local-feed-only packages. |
| S15 | `<PackageReadmeFile>README.md</PackageReadmeFile>` + file present | 🟢 | If declared but absent, `dotnet pack` fails. If absent and not declared, NuGet consumers see no description. |
| S16 | `<PackageIcon>icon.png</PackageIcon>` + file present | 🟡 | Visual identity only. A missing icon doesn't break anything; it's just ugly in NuGet UIs. |
| S17 | `<PackageReleaseNotes>` must be **absent** | 🔴 | Enforcing absence of an optional element is micro-management. Stale release notes are noisy but not harmful. |
| S18 | `icon.png` co-located with `.csproj` | 🟡 | Required for pack to succeed when `<PackageIcon>` is declared. If S16 is kept, this is implied. |

### Exe / non-NuGet projects

| # | Criterion | Rating | Rationale |
|---|-----------|--------|-----------|
| S19 | `<GenerateDocumentationFile>` absent | 🟢 | Present on an exe project produces CS1591 warnings for undocumented members that will never be consumed via NuGet. Noise factory. |
| S20 | NuGet metadata (`PackageId`, etc.) absent | 🟢 | Exe projects are not packages. Metadata here is misleading and can accidentally trigger `dotnet pack`. |
| S21 | `icon.ico` co-located | 🔴 | Windows Explorer icon. Zero functional impact. Cosmetic. |

### Test projects

| # | Criterion | Rating | Rationale |
|---|-----------|--------|-----------|
| S22 | `<GenerateDocumentationFile>` absent | 🟢 | Same reasoning as S19 — test code is not a public API. |
| S23 | NuGet metadata absent | 🟢 | Test projects must never be accidentally published. |
| S24 | `icon.png` / `icon.ico` absent | 🔴 | Icons in test projects have no meaning. Checking their absence is noise. |
| S25 | Required test package refs: `Microsoft.NET.Test.Sdk`, `NUnit 4.x`, `NUnit3TestAdapter` | 🟢 | Without these, `dotnet test` discovers nothing. A broken test project silently passes zero tests. |

---

## 2. Standards — Directory Structure

| # | Criterion | Rating | Rationale |
|---|-----------|--------|-----------|
| S26 | Directory name = assembly name exactly | 🟢 | Navigation by convention. Deviations make the agent (and humans) hunt for files. |
| S27 | `Contracts/` folder present in any project with public static types | 🟡 | Enforces the companion-interface principle. However, small pure-utility assemblies with no public surface may have no interface-worthy types. Risk of false positives. Consider: flag only if public static types exist AND `Contracts/` is absent. |
| S28 | Solution file at repo root | 🟢 | Nested solutions break CI and tooling that assumes root placement. |
| S29 | `nuget.config` present and pointing to `..\..\local-packages\` | 🟢 | Without this, restores hit the internet and local packages are invisible. Build failure guaranteed on a fresh clone. |
| S30 | `.local-build/` scripts present (`build.cmd`, `test.cmd`, `pack.cmd`, etc.) | 🟢 | These are the developer workflow. Missing scripts on a fresh clone = broken onboarding. |

---

## 2. Standards — Naming Conventions

| # | Criterion | Rating | Rationale |
|---|-----------|--------|-----------|
| S31 | Repo name follows `josyn-<layer>` pattern | 🟢 | Consistency; also required by `architecture/overview.md` dependency map. |
| S32 | Namespace follows `JOSYN.<Layer>.<Component>` | 🟢 | Violations break the namespace tree that the whole platform depends on for discoverability. |
| S33 | Assembly name = namespace root | 🟢 | Redundant with A9 but in the standards context. Same rationale. |
| S34 | C# file name = type name | 🟢 | .NET convention; violation confuses every IDE and every human. |
| S35 | Interface file prefixed with `I` | 🟢 | Standard C# convention; violation is immediately visible and confusing. |
| S36 | Error messages in German | 🟡 | Established project convention, documented. Not a functional issue — just consistency. Checking this is hard to automate reliably and somewhat fragile (what counts as an "error message"?). |
| S37 | XML comments and session files in English | 🟡 | Same argument as S36 — meaningful convention but hard to check systematically. |

---

## 3. Docs — XML Documentation

| # | Criterion | Rating | Rationale |
|---|-----------|--------|-----------|
| D1 | `<summary>` required on every documented member | 🟢 | The minimal contract. Without it, `GenerateDocumentationFile` produces no usable output. |
| D2 | `<param>` — omit when name+type are obvious | 🟡 | The "when obvious" clause makes this a judgment call, not a checkable rule. Difficult to enforce without subjective interpretation. |
| D3 | `<returns>` — omit for void / obvious | 🟡 | Same as D2. |
| D4 | `<remarks>` — only when genuinely helpful | 🟡 | The "genuinely helpful" qualifier makes this nearly impossible to check deterministically. |
| D5 | `<exception>` must never be used | 🟢 | Direct mirror of the Result pattern. If `<exception>` appears, the method is either throwing or the doc is wrong — both are violations. |
| D6 | `<inheritdoc/>` on implementations | 🟢 | Without this, doc duplication is inevitable — and duplicated docs drift. |
| D7 | Interfaces carry the full XML docs | 🟢 | Contracts are the API surface. Undocumented interfaces are missing the most important docs. |
| D8 | No duplication of interface doc text in implementations | 🟢 | Duplicated docs will rot at different rates. |
| D9 | `GenerateDocumentationFile=true` on NuGet projects | 🟢 | Checked in standards (S11), but confirming it from the docs side adds a meaningful cross-check. |
| D10 | `GenerateDocumentationFile` absent from exe/test | 🟢 | Same as S19/S22. |
| D11 | Missing `<summary>` on documented member | 🟢 | Catches the case where a `///` block exists but is empty. |
| D12 | Empty `<summary/>` or `// TODO` placeholder | 🟢 | Worse than no comment — actively misleading. |
| D13 | Comment restates method name word-for-word | 🟡 | Bad practice but hard to detect automatically without NLP. An agent can catch obvious cases but will miss subtle ones and flag ambiguous ones. |
| D14 | Comment references renamed type or old namespace | 🟢 | Stale docs are actively misleading and erode trust in all docs. |
| D15 | `<remarks>` with a single obvious sentence | 🟡 | Same problem as D13 — requires subjective judgment. Consider: only flag `<remarks>` that duplicate the `<summary>` exactly. |

---

## 3. Docs — Markdown Documentation

| # | Criterion | Rating | Rationale |
|---|-----------|--------|-----------|
| D16 | `README.md` at repo root | 🟢 | The minimal documentation contract. A fresh clone with no README is an onboarding black hole. |
| D17 | README content accurately reflects current state | 🟡 | High value in principle; almost impossible to check automatically. Requires human judgment about what "current" means. An agent can only flag obvious contradictions. |
| D18 | No stale type/namespace references | 🟢 | Stale references in markdown are directly misleading — a developer follows the README and hits `namespace not found`. |
| D19 | Cross-repo links resolve to existing files | 🟢 | Broken links are a CI-level issue. An agent can follow paths. |

---

## 4. Tests

| # | Criterion | Rating | Rationale |
|---|-----------|--------|-----------|
| T1 | Every public method with realistic failure paths has a test | 🟢 | Core coverage contract. An untested public method is undocumented behaviour. |
| T2 | Positive, negative, edge cases, boundary values | 🟢 | Positive-only tests are systematically blind to failure modes. |
| T3 | Tests verify meaningful behaviour, not line coverage | 🟢 | A test that asserts `Assert.That(true, Is.True)` inflates coverage without value. Flag tests that have no assertion or trivially pass. |
| T4 | Test method naming: `Subject_Scenario_Expected` | 🟢 | A test without a readable name is a test no one can understand when it fails. |
| T5 | Test class/file naming: `<Subject>Tests.cs` | 🟢 | Discoverability. Deviations make the test suite hard to navigate. |
| T6 | Arrange-Act-Assert structure | 🟡 | Good practice but the blank-line separation rule is cosmetic. The structural pattern is what matters. |
| T7 | Fluent assertions: `Assert.That(x, Is.EqualTo(y))` — not classic `Assert.AreEqual` | 🟡 | Style preference within NUnit. Both work identically. Enforcing this is cosmetic. |
| T8 | One observable outcome per test | 🟢 | Multiple independent assertions in one test make failures ambiguous and reduce information density. |
| T9 | No flow control (if/loop) in test methods | 🟢 | A test with conditional logic can silently skip assertions. High-value rule. |
| T10 | Tests are independent — no shared mutable state | 🟢 | Order-dependent tests are a reliability nightmare in CI. |
| T11 | `[TestCase]` for repeated patterns | 🟡 | Strongly encouraged but not critical. Copy-pasted tests work; they're just verbose. |
| T12 | `Assert.Throws<T>` / `Does.Throw` for exception testing | 🟢 | The alternative `[ExpectedException]` is removed in NUnit 3+; presence indicates stale code. |
| T13 | No `[ExpectedException]` attribute | 🟢 | Outdated, removed API. Presence indicates unmaintained tests. |
| T14 | Same result on every run (determinism) | 🟢 | Non-deterministic tests are worse than no tests — they destroy confidence in the suite. |
| T15 | File system / clock / network tests in separate assembly or marked explicitly | 🟢 | An integration test disguised as a unit test is a CI time bomb. |
| T16 | No `[Ignore]` without reason string | 🟢 | Undocumented ignore = silent dead test. Nobody knows why or for how long. |
| T17 | No permanently ignored tests | 🟢 | A forever-ignored test is a lie in the test suite. Delete it or fix it. |
| T18 | Test double guidance (stub before mock, no over-mocking) | 🟡 | Good advice. "Over-mocked" is a judgment call — hard to operationalise as a pass/fail check. Consider: flag tests where every collaborator is replaced with a mock. |
| T19 | Committed test that fails | 🟢 | Non-negotiable. A failing test in the repository is a broken contract. |

---

## 5. Principles

| # | Criterion | Rating | Rationale |
|---|-----------|--------|-----------|
| P1 | Instance class with no state and no polymorphism need → must be `static` | 🟢 | Core philosophy. An unnecessary instance class is a design smell that misleads the reader about purpose. |
| P2 | `class` used where `record` would suffice for a data carrier | 🟢 | A mutable data class is an accidental mutation risk. The check enforces a concrete, checkable rule. |
| P3 | Mutable property (`{ get; set; }`) without justification | 🟢 | Undocumented mutability is the most common source of subtle bugs in the codebase. |
| P4 | `throw` anywhere above the lowest-layer catch boundary | 🟢 | Any `throw` above the boundary breaks the Result contract. Straightforward to detect. |
| P5 | `catch` that swallows (no conversion to `Result`) | 🟢 | A swallowed exception is a hole in the error model — failures disappear silently. |
| P6 | Method that can fail returns `void` or raw value instead of `Result`/`Result<T>` | 🟢 | Same contract violation — just the return-site perspective of P4/P5. |
| P7 | Manual re-wrap of `Result` instead of `Result.Propagate(inner)` | 🟢 | Re-wrapping loses the call chain. Propagate preserves context — enforcing it is cheap and high-value. |
| P8 | Public static type without companion interface in `Contracts/` | 🟡 | Valid for the main public API surface. However, small internal utility statics don't need an interface. Risk of false positives. Scope should be: **public static types only**. |
| P9 | Interface used for polymorphism rather than as a shape contract | 🟡 | The rule makes sense but checking it requires distinguishing *intent* — hard to do statically. An agent can flag interfaces that are implemented by more than one non-test class as candidates, but it's inherently a judgment call. |
| P10 | DI container wiring present | 🟢 | Any DI container reference is unambiguous and violates the explicit-over-magic principle directly. Easy to check (`IServiceCollection`, `IServiceProvider`, `AddSingleton`, etc.). |
| P11 | Reflection-based wiring outside the designated `[JobEntryPoint]` dispatch | 🟢 | Reflection-based magic is exactly what the platform explicitly forbids. The exception is tightly scoped and documented. |
| P12 | Hidden conventions (auto-discovery, convention-over-configuration outside designed entry points) | 🟡 | High value but vague scope. "Convention-over-configuration" is a fuzzy concept — hard to operationalise without specific signals. Consider: narrow to `Assembly.GetTypes()`, `Activator.CreateInstance`, `GetCustomAttributes`. |
| P13 | `public` member where `internal` would suffice | 🟢 | Minimal surface area is directly enforceable. An oversized public API locks future changes. |
| P14 | Method mutates state AND returns a result without documenting the side effect | 🟡 | Important in principle but requires the agent to reason about whether mutation is documented, which is subjective. Consider: flag methods with both a return value and an `out`/`ref` parameter or a field mutation that have no `<remarks>` explaining the side effect. |

---

## Summary by Rating

| Rating | Count | Notes |
|--------|-------|-------|
| 🟢 Indispensable | 46 | Keep exactly as written |
| 🟡 Discussable | 20 | Review each individually — may need scoping, weakening, or a note about the judgment-call nature |
| 🔴 Not really necessary | 6 | Consider removing or demoting to "advisory" |
| ⛔ Bare mistake | 0 | No criterion is actively wrong — the set is clean in structure |

### 🔴 Candidates for removal (6)

| ID | Criterion | Removal rationale |
|----|-----------|-------------------|
| S7 | `IncludeSourceRevisionInInformationalVersion=false` | Version metadata noise; no practical impact |
| S8 | Indentation = tabs in csproj | Reformatted on save; cosmetic-only noise |
| S9 | Single `<PropertyGroup>` | Cosmetic; multiple groups are sometimes clearer |
| S17 | `<PackageReleaseNotes>` must be absent | Micro-management of an optional metadata element |
| S21 | `icon.ico` co-located with exe `.csproj` | Windows Explorer cosmetic; zero functional impact |
| S24 | `icon.png/ico` absent from test projects | No icons in test projects is obvious; not worth a check |

### 🟡 Criteria needing scoping discussion (selected highlights)

| ID | Concern | Suggested resolution |
|----|---------|---------------------|
| A8 | Named pipe pattern — hard to check statically | Limit to: flag hardcoded strings that look like pipe names but deviate from the pattern |
| S27 | `Contracts/` folder — false positives on tiny utility assemblies | Scope to: public static types exist AND `Contracts/` is absent |
| P8 | Companion interface — same as S27 | Scope to **public** static types only, not internal |
| P9 | Interface for polymorphism | Change from a check to an advisory note (flag multiple implementations as candidates for review) |
| P12 | Hidden conventions | Narrow to specific detectable signals: `Assembly.GetTypes()`, `Activator.CreateInstance`, `GetCustomAttributes` |
| T18 | Over-mocking | Change from a check to advisory; flag only when >3 mocks AND no real collaborators in a test |
| D13 | Comment restates name | Limit to exact match only (e.g. `public void Run()` with `<summary>Run</summary>`) |
| D15 | `<remarks>` with obvious sentence | Narrow to: `<remarks>` content = `<summary>` content (exact duplication only) |

---

## Review History

| Date | Reviewer | Changes made |
|------|----------|--------------|
| 2026-05-30 | Initial draft | First complete pass over all 72 criteria |
