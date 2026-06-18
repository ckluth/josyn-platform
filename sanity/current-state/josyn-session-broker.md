# Sanity State — josyn-session-broker

**Last checked:** 2026-06-18T19:27 UTC

## Summary

| Category | Status |
|----------|--------|
| architecture | ✅ |
| standards | ✅ |
| docs | ❌ 1 violation |
| tests | ⚠️ known gap |
| principles | ❌ 3 violations + 1 minor |

---

## architecture

✅ All NuGet dependency edges are legitimate per ADR-025 (`josyn-commons`, `josyn-backend`,
  `josyn-adapter-contracts`, `josyn-foundation`, `josyn-jap` — all permitted).
✅ No forbidden cross-repo project references — all dependencies are package references.
✅ Named-pipe session key derived from GUID (`sessionGuid` passed to `PipesServer.RunAsync`).

ℹ️ Note: `sanity/criteria/architecture.md` still references `JOSYN.Jap.JAPServer` and
  `josyn-backend` as the host of the JAPServer EXE. These entries are stale post-ADR-025.
  This is a criteria-file defect, not a josyn-session-broker violation. Tracked separately.

---

## standards

✅ `<OutputType>Exe</OutputType>` — correct for EXE.
✅ `<TargetFramework>net10.0</TargetFramework>`
✅ `<Nullable>enable</Nullable>`, `<LangVersion>latest</LangVersion>`
✅ `<IncludeSourceRevisionInInformationalVersion>false</IncludeSourceRevisionInInformationalVersion>`
✅ `<AppendTargetFrameworkToOutputPath>false</AppendTargetFrameworkToOutputPath>`
✅ `<PlatformTarget>AnyCPU</PlatformTarget>`
✅ `icon.ico` co-located and referenced via `<ApplicationIcon>` and `<Content Include>`.
✅ No `<GenerateDocumentationFile>` (correct for EXE).
✅ No NuGet packaging metadata (`<PackageId>`, `<PackageLicenseExpression>`, `<PackageTags>` absent).
✅ `nuget.config` points to `..\local-packages` — correct for a Pattern A repo
  (one level up from `C:\DevGit\josyn-session-broker\` reaches `C:\DevGit\local-packages\`).
✅ `.local-build\` scripts present (build.cmd, pack.cmd, clean.cmd, etc.).
✅ Solution file `JOSYN.SessionBroker.slnx` at repo root.
✅ Namespace `JOSYN.SessionBroker` — two-segment form, intentional per ADR-025 (same pattern
  as `JOSYN.JobHost`).

ℹ️ Known exception: directory name `JOSYN.SessionBroker` differs from `<AssemblyName>SessionBroker`.
  This is intentional — the EXE output must be `SessionBroker.exe` (not `JOSYN.SessionBroker.exe`).
  Documented in ADR-025. Not a violation.

---

## docs

❌ `JOSYN.SessionBroker/AdapterManager.cs`
   Class `<summary>` says "JAPServer session" — stale post-ADR-025 rename.
   Should read "SessionBroker session".

✅ `AdapterProcess.cs` — class `<summary>` present and accurate.
✅ `AdapterManager.GetPipes` — `<summary>` present and accurate.
✅ `Host.Adapters.cs` — `SpawnAdapters` has full `<summary>` with `<paramref>` and `<see cref>`.
✅ All partial-class `Host.*` methods with non-obvious behaviour have inline `//` comments.
✅ `PrepareContext`, `JapInfrastructure`, `NegotiationOutcome` records all commented.
✅ `TerminalStatusSet` — has inline block comment explaining the double-write guard.
✅ `ServerTask!` null-forgiving usage has a justification block comment.
✅ `README.md` at repo root — present and accurate.

---

## tests

⚠️ No test project exists. This is a known gap — the EXE hosts integration-level behaviour
  that is difficult to unit test in isolation. Documented in `repos/josyn-session-broker.md`
  under Sanity Notes as expected state. Not treated as a violation.

---

## principles

❌ P1 — `JOSYN.SessionBroker/Program.cs` line 3
   `internal class Program` has no instance members — only a `static Main` method.
   Must be `internal static class Program` per P1 (static wins).

❌ P8 — `JOSYN.SessionBroker/AdapterProcess.cs` line 18
   `PipesClient.DisconnectAsync(...).GetAwaiter().GetResult()` in `Dispose()`.
   `.GetAwaiter().GetResult()` on a Task is a P8 violation.
   Fix: implement `IAsyncDisposable` on `AdapterProcess` and `AdapterManager`;
   update `Host.Run` to `await using var adapterManager`.

❌ P10 — `JOSYN.SessionBroker/Host.Entrypoint.cs` lines 120 and 126
   `ctx.SessionBroker!` used twice without an inline justification comment.
   `ctx.ServerTask!` at line 111 has a justification block comment; `SessionBroker!` does not.
   Fix: extend the existing block comment (lines 105–110) to also explain why `SessionBroker`
   is non-null at that point (it is set in `BuildJapInfrastructure`, which must have succeeded
   for `NegotiationAccepted` to be true — the guard at line 102 ensures this).

ℹ️ Minor P8 — `JOSYN.SessionBroker/Host.Server.cs` line 21
   `HandleHandlerError` is declared `async Task` but only calls `errorHandler.Handle` (sync)
   and then `await Task.CompletedTask` (no-op). The `async` state machine is unnecessary.
   Fix: remove `async`, return `Task.CompletedTask` directly.

✅ P2 — `PrepareContext`, `JapInfrastructure`, `NegotiationOutcome` are immutable `sealed record`s.
  `AdapterProcess` properties are init-only (`{ get; }`). No unjustified mutable fields.
✅ P3 — No `throw` outside catch boundaries. All failure paths return `Result` / `Result<T>`.
  `catch { /* ignore */ }` at cleanup boundaries (File.Delete, Process.Kill, Dispose) — correct.
✅ P4 — `SessionBroker` implements `IJosynApplicationProtocol` via explicit interface implementation.
  No public type lacks a companion interface.
✅ P5 — No DI containers, no reflection-based dispatch, no attribute magic.
✅ P6 — All types are `internal` (except `IDisposable.Dispose()` required by interface).
✅ P7 — Predictable control flow throughout. Early returns at each failure point.
✅ P9 — Methods are short and well-named. Each partial file owns one coherent concern.
  Nested helpers placed after `return`, happy path on top.
