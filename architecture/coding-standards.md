# Coding Standards — Functional-First C#

> **Philosophy:** Functional-first C#. Not F#, but C# written *as if it could be*.
> Every design choice is measured against: "Would this pattern translate naturally into a functional language?"

---

## Principles (ranked by priority)

### 1. Static wins (when in doubt)

Prefer `static class` and `static` methods over instance methods.

Instance types and OOP patterns are *tools*, pulled in only when they earn their place:
- a natural identity
- mutable state that needs encapsulation
- a genuine polymorphism need

*"In doubt — static wins"* is the tiebreaker.

```csharp
// Preferred
public static class JobInvoker
{
    public static async Task<Result> InvokeJob(...) { ... }
}

// Only if mutable state or polymorphism is genuinely needed
public sealed class JAPClient : IJosynApplicationProtocol { ... }
```

### 2. Immutability by default

Prefer `record` over `class`. Prefer `readonly` fields and `init`-only properties.
Mutable state must be explicitly justified — never the path of least resistance.

```csharp
// Preferred
public sealed record ErrorReport(
    string Causer,
    string Message,
    string? CallStack,
    string? ExceptionDetails,
    DateTimeOffset OccurredAt);

// Mutable state only when justified
public static string LogDirectory { get; set; }   // LocalLog — process-global config
```

### 3. Pure functions over side effects

A method that does not mutate state and returns a value encoding its full result is the ideal.
Side effects are isolated, named explicitly, and pushed to the edges of the call graph.

### 4. Errors as values, never exceptions

`Result` / `Result<T>` is the single mechanism for propagating failures.

```csharp
// Correct
public static Result<Assembly> FindEntryPointAssembly(Type? hint = null)
{
    try
    {
        var asm = hint != null ? Assembly.GetAssembly(hint) : Assembly.GetEntryAssembly();
        return asm ?? Result.Error("Assembly nicht gefunden.");
    }
    catch (Exception ex) { return ex; }  // ← bottom-layer catch: converts to Result
}

// Wrong — never propagate exceptions
public static Assembly FindEntryPointAssembly() { ... }  // throws
```

**Propagation rule:** use `Result.Propagate(inner)` to accumulate the call chain. Never re-wrap manually.

```csharp
var find = FindEntryPointAssembly();
if (!find.Succeeded)
    return Result.Propagate(find.ToResult());  // ✓ preserves call chain
```

**Implicit conversions** keep happy-path code concise:

```csharp
// All valid:
return asm;                          // Assembly → Result<Assembly>
return new Error("Fehlermeldung");   // Error → Result<Assembly>
return ex;                           // Exception → Result<Assembly>
return Result.Success;               // void success
```

> **Ternary trap — `Result<string>`:** When `TValue = string`, never rely on the implicit
> `string → Result<string>` conversion as a ternary branch when the other branch is `Error`.
> The C# compiler resolves the common type of both branches — not the target type — and will
> silently convert the `string` to `Error(string)` (a failure). Always use
> `Result<string>.Success(value)` explicitly on the success branch:
>
> ```csharp
> // WRONG — "INT" silently becomes Error("INT"):
> Result<string> GetValue(string key) =>
>     dict.TryGetValue(key, out var v) ? v : Result.Error("not found");
>
> // CORRECT:
> Result<string> GetValue(string key) =>
>     dict.TryGetValue(key, out var v) ? Result<string>.Success(v) : Result.Error("not found");
> ```
>
> `implicit operator Error(string)` has been deliberately removed from `Error` to prevent
> this class of bug. `string → Error` no longer compiles.

### 5. Interfaces as contracts, not polymorphism

Public static types get a companion interface with `static abstract` members in `Contracts/`.
These interfaces are API documentation and shape contracts, not polymorphism points.

```csharp
// Contracts/IPropertyBag.cs
public interface IPropertyBag
{
    static abstract Result<string> Serialize<TRecord>(TRecord record)
        where TRecord : class;
    // ...
}

// PropertyBag.cs
public static class PropertyBag : IPropertyBag
{
    /// <inheritdoc cref="IPropertyBag.Serialize{TRecord}"/>
    public static Result<string> Serialize<TRecord>(TRecord record) { ... }
}
```

### 6. Explicit over magic

No reflection-based wiring (except the deliberately designed `[JobEntryPoint]` dispatch), no DI containers, no hidden conventions.

### 7. Minimal surface area

A type should expose only what is needed. Internal types stay internal.

### 8. Strict access modifiers

Be as strict as possible. Expose only what must be exposed.
Prefer `internal` over `public` wherever the wider API surface is not required.

### 9. Readable structure — short methods, named groups, partial classes

**The overview and flow must be graspable at a glance.** This is the dimension that is most
often sacrificed when code is generated — technically correct structure that is opaque to a
human reader is a maintenance liability from the first commit.

**Short methods.** No method should be so long that its internal structure is not immediately
apparent. If a method contains distinct logical phases — parse, validate, execute, return —
each phase is a candidate for extraction.

**Named logical groups.** When extracting, prefer pure functions and immutable intermediates.
Name extracted methods for *what they do*, not *how they do it*.

**Nested helpers.** If a helper only has meaning inside one super-method, place it as a
nested local function *after the `return`* in that method. The happy path stays on top; the
mechanics go below — out of the way, but right where they belong.

```csharp
private static async Task<int> Main(string[] args)
{
    if (!TryParseSessionGuid(args, out var guid))
        return FailWith("Usage: ...");

    var result = await RunServer(guid);
    return result.Succeeded ? 0 : 1;

    // ── helpers ───────────────────────────────────────────────────────
    static bool TryParseSessionGuid(string[] args, out Guid guid) { ... }
    static int  FailWith(string message) { ... }
}
```

**Partial classes for logical steps.** When a class contains many methods that naturally
group into lifecycle phases or responsibility areas, split them into partial class files
using a `.`-separated naming pattern:

```
Host.Entrypoint.cs
Host.Adapters.cs
Host.Prepare.cs
Host.Negotiation.cs
```

Each file owns one coherent concern. A reader opening any one file immediately knows its
scope — and the full picture emerges from the file listing alone.

**Comments.** A short comment on anything that is not crystal-clear is more than
appreciated. Comments explain *why*, not *what*. A comment that restates the code adds
noise; a comment that explains a non-obvious constraint or a tradeoff saves the next reader
from having to reconstruct the reasoning from scratch.

### 10. No mutation of reference-type parameters

Never modify an object that was passed into a method as a reference-type parameter.
A caller who passes a reference does not expect the callee to alter its state.
If a modified version is needed, produce and return a new instance.

This rule reinforces Principle 2 (Immutability by default) at the method-boundary level.
Immutability is not just about field declarations — every method signature is a contract,
and mutating an incoming reference silently violates that contract from the caller's perspective.

```csharp
// ❌ Wrong — mutates the caller's object without any indication in the signature
private static void EnrichContext(ExecutionContext ctx)
{
    ctx.StartedAt = DateTimeOffset.UtcNow;   // caller's object is silently changed
}

// ✅ Correct — return a new instance; the caller decides what to do with it
private static ExecutionContext WithStartTime(ExecutionContext ctx) =>
    ctx with { StartedAt = DateTimeOffset.UtcNow };
```

When a reference-type parameter genuinely must carry a mutation (e.g., a builder or
accumulator pattern), that intent must be explicit in the method name and documented
in the XML summary. Even then, prefer returning a new value over mutating the input.

### 11. `job.exe` must not rely on a user profile (ADR-021)

`job.exe` is launched with `LoadUserProfile = false` — the technical user account has no
local profile on the execution machine. The following environment variables are
**not available** to a running job:

| Unavailable | Use instead |
|---|---|
| `%APPDATA%` | — |
| `%LOCALAPPDATA%` | — |
| `%USERPROFILE%` | — |
| `%TEMP%` (user-scoped) | `%TEMP%` (machine-scoped, set by the OS for services) |

System-scoped paths are safe: `%ProgramData%`, `%SystemRoot%`, `%WINDIR%`.

Any job that reads or writes files must use absolute paths, paths derived from arguments,
or system-scoped environment variables — never profile-relative paths.

---

## Agent Application Guide

### Apply automatically — without being asked

- When proposing a new type: start with `static class` and state why if choosing something else.
- When reviewing existing code: flag instance types that have no state and could be `static`.
- When writing XML docs: put them on the interface, not the implementation.
- When a failure path exists: return `Result.Error(...)` or `return ex;` — never `throw`.
- When propagating a failure upward: use `Result.Propagate(inner)` — never re-wrap.
- When choosing an access modifier: default to `internal`; escalate to `public` only when the member is part of the intended public API.
- When writing any method: keep it short enough that its structure is apparent at a glance. If distinct logical phases exist (parse / validate / execute / return), extract each phase into a named helper.
- When extracting helpers: prefer pure functions and immutable intermediates. Name for *what*, not *how*.
- When a helper only serves one super-method: place it as a nested local function *after the `return`* in that method — happy path first, mechanics below.
- When a class grows several methods that cluster into lifecycle phases or responsibility areas: split into partial class files with a `.`-separated naming pattern (`Host.Entrypoint.cs`, `Host.Adapters.cs`, …). Each file owns one coherent concern.
- When anything in the code is not crystal-clear: add a short inline comment explaining *why*. Never comment what the code already says.
- When writing a method that receives a reference-type parameter: never mutate it. If a modified version is needed, construct and return a new instance. Name the method to reflect the transformation (e.g., `WithStartTime`, not `EnrichContext`).
- When calling `IErrorHandler.Handle()` and a failed `Result` is in scope at the call site: use the `Handle(Result, ...)` overload — it extracts all diagnostic context automatically. The raw `Handle(string, string?, string?, ...)` overload is for paths where no `Result` exists; passing `null, null` there is correct and deliberate. Using the raw overload when a `Result` is available is a violation.
- When calling `IErrorHandler.Handle()`: do not also call `LocalLog` at the same site. The handler's own fallback writes to `LocalLog` if SQL fails — calling both unconditionally makes `LocalLog` a co-primary, not a fallback. The only legitimate `LocalLog` call sites outside `SqlErrorHandler` itself are bootstrap paths that execute before the error handler is constructed.

### Never do

- Add OOP abstraction layers "just in case" future flexibility is needed.
- Reach for dependency injection when a static call or a delegate parameter suffices.
- Create mutable state when an immutable data pipeline would work.
- Write defensive `try/catch` blocks outside the lowest-layer boundary.
- Use `public` where `internal` is sufficient.
- Use the null-forgiving operator (`!`) without an inline comment that proves the value cannot be null — prefer eliminating it via Result propagation entirely.
- Mutate a reference-type parameter passed into a method — create a new instance instead.
- Call `LocalLog.WriteError(...)` and `errorHandler.Handle(...)` at the same error site — that is double-logging. `LocalLog` is the fallback *inside* the handler; call sites must not duplicate it. The no-swallow guarantee in ADR-006 §6 proves the fallback chain is closed without call-site involvement.

---

## Code Style

| Topic | Preference |
|-------|-----------|
| Class kind | `static class` by default; `record` for data; `class` only when mutable state or OOP is needed |
| Nullability | `Nullable=enable` everywhere; `?` is deliberate, never defensive; `!` requires an inline justification comment — eliminate it via Result propagation where possible |
| Error handling | `Result` / `Result<T>` — no `throw`, no `try/catch` above the lowest layer |
| Comments | Only where the *why* is non-obvious; no restating what the code already says |
| XML docs | On interfaces/contracts; implementations use `<inheritdoc>` |
| `<remarks>` | Used for "not yet implemented" placeholders on future-planned attributes |
| Pragmas | `#pragma warning disable IDE0130` used only to suppress namespace-path mismatch warnings in flat-namespace projects |

---

## Result Pattern — Quick Reference

```csharp
using JOSYN.Foundation.ResultPattern;

// Void operation
Result DoSomething()
{
    if (failed) return Result.Error("Fehlermeldung");
    return Result.Success;
}

// Typed operation
Result<string> GetSomething()
{
    if (failed) return new Error("Fehlermeldung", optionalException);
    return "the value";    // implicit conversion
}

// Propagate upward (preserves call chain)
var inner = DoSomething();
if (!inner.Succeeded)
    return Result.Propagate(inner);

// Inspect
if (!result.Succeeded)
{
    Console.WriteLine(result.ErrorMessage);
    Console.WriteLine(result.CallStackAsString);
}
```

**Key types:**

| Type | Role |
|------|------|
| `Result` | Void operation result |
| `Result<T>` | Typed operation result |
| `Error` | Value-type error carrier (`string` + `Exception?`) |
| `CallerInfo` | One entry in the propagation chain |
| `IResult<TSelf>` | Static abstract contract for `Result` |
| `IResult<TSelf,T>` | Static abstract contract for `Result<T>` |

---

## Project File Conventions

Every NuGet-publishable project includes:

```xml
<PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <LangVersion>latest</LangVersion>
    <Version>1.0.0-preview01</Version>
    <Company>HAEVG</Company>
    <Authors>HAEVG SWE</Authors>
    <PackageLicenseExpression>MIT</PackageLicenseExpression>
    <Copyright>Copyright © 2026 HAEVG AG</Copyright>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <AppendTargetFrameworkToOutputPath>false</AppendTargetFrameworkToOutputPath>
    <IncludeSourceRevisionInInformationalVersion>false</IncludeSourceRevisionInInformationalVersion>
</PropertyGroup>
```

---

## NuGet Feed

All packages are resolved from a local feed at `..\..\local-packages\` (relative to each repo root).
Defined in each repo's `nuget.config`.

Build + pack workflow via `.local-build\pack.cmd` → outputs to the local feed.
Consuming repos then pick up the new version on restore.
