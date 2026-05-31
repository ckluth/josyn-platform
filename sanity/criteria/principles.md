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
