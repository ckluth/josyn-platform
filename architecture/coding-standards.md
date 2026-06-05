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

---

## Agent Application Guide

### Apply automatically — without being asked

- When proposing a new type: start with `static class` and state why if choosing something else.
- When reviewing existing code: flag instance types that have no state and could be `static`.
- When writing XML docs: put them on the interface, not the implementation.
- When a failure path exists: return `Result.Error(...)` or `return ex;` — never `throw`.
- When propagating a failure upward: use `Result.Propagate(inner)` — never re-wrap.
- When choosing an access modifier: default to `internal`; escalate to `public` only when the member is part of the intended public API.
- When calling `IErrorHandler.Handle()` and a failed `Result` is in scope at the call site: use the `Handle(Result, ...)` overload — it extracts all diagnostic context automatically. The raw `Handle(string, string?, string?, ...)` overload is for paths where no `Result` exists; passing `null, null` there is correct and deliberate. Using the raw overload when a `Result` is available is a violation.

### Never do

- Add OOP abstraction layers "just in case" future flexibility is needed.
- Reach for dependency injection when a static call or a delegate parameter suffices.
- Create mutable state when an immutable data pipeline would work.
- Write defensive `try/catch` blocks outside the lowest-layer boundary.
- Use `public` where `internal` is sufficient.

---

## Code Style

| Topic | Preference |
|-------|-----------|
| Class kind | `static class` by default; `record` for data; `class` only when mutable state or OOP is needed |
| Nullability | `Nullable=enable` everywhere; `?` is deliberate, never defensive |
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

`Directory.Build.props` at repo root redirects all build output to `C:\Temp\VS.OUT\JOSYN\<ProjectName>\` (PoC convenience — replace with standard paths for CI).

---

## NuGet Feed

All packages are resolved from a local feed at `..\..\local-packages\` (relative to each repo root).
Defined in each repo's `nuget.config`.

Build + pack workflow via `.local-build\pack.cmd` → outputs to the local feed.
Consuming repos then pick up the new version on restore.
