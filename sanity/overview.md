# Sanity Check — Platform Overview

| Repo | docs | tests | principles | architecture | standards | Last checked |
|------|------|-------|-----------|-------------|-----------|-------------|
| josyn-foundation | ✅ | ✅ | ❌ 5+3 violations | ❌ 1 violation | ✅ | 2026-05-31T11:41 UTC |
| josyn-jap | — | — | — | — | — | never |
| josyn-job-host | — | — | — | — | — | never |
| josyn-backend | — | — | — | — | — | never |
| josyn-commons | — | — | — | — | — | never |

---

## josyn-foundation — Violation Summary

### principles ❌

**P1 Static wins (5 violations):** Types with zero instance state declared as `sealed class` instead of `static class`:
- `PropertyBag` — `josyn-foundation-property-bag/JOSYN.Foundation.PropertyBag/PropertyBag.cs`
- `PipesClient` — `josyn-foundation-jip/JOSYN.Foundation.JIP/PipesClient.cs`
- `PipesServer` — `josyn-foundation-jip/JOSYN.Foundation.JIP/PipesServer.cs`
- `PipesProtocol` — `josyn-foundation-jip/JOSYN.Foundation.JIP/PipesProtocol.cs`
- `JipProtocol` — `josyn-foundation-jip/JOSYN.Foundation.JIP/Jip/JipProtocol.cs`

**P3 No throw (3 violations):**
- `Result.ToResult<TValue>()` — throws `InvalidOperationException` on misuse
- `Result<T>.ToResult<TOther>()` — same
- `JipDispatcher.RegisterAll` — throws `InvalidOperationException` for unsupported method signatures

### architecture ❌

**1 violation:** Pipe names `"req-pipe-{key}"` / `"res-pipe-{key}"` deviate from the documented `JOSYN-REQ-<GUID>` / `JOSYN-RSP-<GUID>` pattern in `architecture/overview.md`. The code and its interface contract are internally consistent; the architecture doc needs alignment. Decide: update doc or update implementation.
