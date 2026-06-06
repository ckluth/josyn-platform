# Sanity Criteria — principles

> Verify the codebase applies the functional-first C# coding principles.
> Read `architecture/coding-standards.md` before evaluating — it contains
> the full principle definitions, rationale, and code examples.

---

## Checklist

Evaluate each item for every type and method in scope.

### 1. Static wins

| Signal | Verdict |
|--------|---------|
| Instance class with no instance state and no polymorphism need | ❌ violation |
| Instance class that could trivially be `static` | ❌ violation |
| `static class` used for stateless logic | ✅ pass |

### 2. Immutability

| Signal | Verdict |
|--------|---------|
| `class` used where `record` would suffice for a data carrier | ❌ violation |
| Mutable property (`{ get; set; }`) without documented justification | ❌ violation |
| `record` or `readonly` field for data types | ✅ pass |

### 3. Errors as values — never exceptions

| Signal | Verdict |
|--------|---------|
| `throw` statement anywhere above the lowest-layer catch boundary | ❌ violation |
| `try/catch` block that swallows exceptions (no conversion to `Result`) | ❌ violation |
| Method that can fail but returns `void` or a raw value instead of `Result`/`Result<T>` | ❌ violation |
| Exception manually re-wrapped instead of using `Result.Propagate(inner)` | ❌ violation |
| `catch` at lowest layer converts `Exception` to `Result` (i.e. `return ex;`) | ✅ pass |
| Failure propagated upward via `Result.Propagate(inner)` | ✅ pass |

### 4. Interfaces as contracts

| Signal | Verdict |
|--------|---------|
| Public static type without a companion interface in `Contracts/` | ❌ violation |
| Public method signature uses a concrete type where the companion interface exists and should be used (e.g. `ServerStartArguments` instead of `IServerStartArguments` as a parameter) | ❌ violation |
| Interface implemented by more than one non-test class | ⚠️ candidate — review whether intent is polymorphism vs. shape contract |
| Implementation duplicates interface doc instead of using `<inheritdoc/>` | ❌ violation |
| `static abstract` members on companion interface match the static class surface | ✅ pass |

### 5. Explicit over magic

| Signal | Verdict |
|--------|---------|
| DI container wiring present | ❌ violation |
| Reflection-based wiring not part of the deliberately designed `[JobEntryPoint]` dispatch | ❌ violation |

### 6. Minimal surface area / strict access modifiers

| Signal | Verdict |
|--------|---------|
| `public` member where `internal` would suffice | ❌ violation |
| Internal type accidentally exposed as `public` | ❌ violation |
| Only the intended API surface is `public` | ✅ pass |

### 7. Pure functions over side effects

| Signal | Verdict |
|--------|---------|
| Method mutates a field AND returns a value, with no `<remarks>` documenting the side effect | ❌ violation |
| Side-effecting code pushed to the call graph edges | ✅ pass |

### 8. Async correctness

| Signal | Verdict |
|--------|---------|
| `async void` method (outside event handler) | ❌ violation |
| `.Result` property or `.GetAwaiter().GetResult()` call on a `Task` | ❌ violation |

### 10. Null-forgiving operator (`!`)

`Nullable=enable` is mandatory everywhere. The null-forgiving operator `!` suppresses a
compiler safety net — it is the nullability equivalent of a bare `throw`: it converts a
compile-time guarantee into a runtime gamble.

| Signal | Verdict |
|--------|---------|
| `!` used without an inline comment that proves the value cannot be null at that point | ❌ violation |
| `!` used on a `Result<T>.Value` access without first checking `.Succeeded` | ❌ violation — use the Result pattern; access `.Value` only inside a guarded branch |
| `!` used with a clear, specific justification comment immediately on the same line or the line above | ✅ pass — the author has accepted responsibility and documented why |

**Examples:**

```csharp
// ❌ violation — silent assumption
var dir = Path.GetDirectoryName(path)!;

// ❌ violation — undocumented .Value! dereference
var config = FileBootstrapConfig.Load(path).Value!;

// ✅ pass — justification is present and specific
var dir = Path.GetDirectoryName(path)!; // path passed File.Exists() above — GetDirectoryName cannot return null for a valid file path

// ✅ preferred — eliminate the ! entirely via Result propagation
var loadResult = FileBootstrapConfig.Load(path);
if (!loadResult.Succeeded)
    return Result.Propagate(loadResult.ToResult());
var config = loadResult.Value;
```

When reviewing: prefer eliminating `!` over justifying it. A justified `!` is a last resort,
not a coding style.

`IErrorHandler` has two overloads by design. The compiler enforces that `callStack` and
`exceptionDetails` are always provided explicitly on the raw path. What the compiler cannot
enforce is *which overload* to use. That is a code-review responsibility.

| Signal | Verdict |
|--------|---------|
| `Handle(result, ...)` used when a failed `Result` is in scope | ✅ pass — preferred path |
| `Handle(msg, null, null)` when no `Result` or `Exception` exists at the call site | ✅ pass — honest acknowledgement |
| `Handle(msg, null, null)` or `Handle(msg, null, ex.ToString())` when a failed `Result` is in scope | ❌ violation — use the `Result` overload; raw-string call discards the call chain |
| `Handle(msg, callStack: null, exceptionDetails: ex.ToString())` when a caught `Exception` is in scope and no `Result` exists | ✅ pass |

**Why:** The `Result`-typed overload extracts `ErrorMessage`, `CallStackAsString`, and
`Exception` automatically. Using the raw overload when a `Result` is available is a manual
re-extraction that is both more verbose and prone to forgetting `CallStackAsString`.
The compiler prevents the worst case (bare call); this rule prevents the subtle case.
