# ADR-003 — josyn-commons: A Generic Utility Layer

**Date:** 2026-05-30
**Status:** Accepted

---

## Context

As the JOSYN platform matures, recurring utility needs arise across multiple repos — small,
domain-agnostic helpers that have no natural home in any single consuming repo, but do not
belong in `josyn-foundation` either.

`josyn-foundation` is a **stable-forever** infrastructure layer: its contracts are load-bearing
and intentionally frozen. Placing generic, evolving utilities there would erode that guarantee
and blur the semantic of "foundation".

A new layer is needed that:

1. Sits at the **bottom of the dependency graph** — referenced by any repo, referenced by none.
2. Is **open for growth** — new helpers are added over time as common patterns emerge.
3. Is **always backward-compatible** — existing consumers are never broken.
4. Is **not a first-class platform citizen** — it carries no protocol knowledge, no topology
   awareness, and no JOSYN-domain logic.

---

## Decision

Introduce a new repo **`josyn-commons`** with namespace root **`JOSYN.Commons.*`**.

### Name rationale

Three candidates were evaluated against the .NET developer audience:

| Candidate | Familiarity | Semantic clarity | Risk |
|-----------|-------------|-----------------|------|
| `utils` | Highest — ubiquitous in .NET | Low — implies a junk drawer | Sounds disposable |
| `toolkit` | Medium | Medium — implies curation | Leans toward developer tooling / SDK |
| `commons` | Medium — known from Apache Commons | High — "shared pool owned by all" | Slight Java connotation |

**`commons` was chosen** because it names the architectural *role* ("shared pool at the bottom,
owned by no layer, used by all"), not merely the content. The Apache Commons heritage is an
asset for .NET developers who recognise it — they immediately understand the intent.

### Architectural position

```
JOSYN.Commons.*  (no dependencies, or ResultPattern only)
      ▲  ▲  ▲  ▲
      │  │  │  │
   (any josyn repo may reference it)
```

`josyn-commons` must never take a dependency on `PropertyBag`, `JIP`, `josyn-jap`,
`josyn-job-host`, or `josyn-backend`. Only `JOSYN.Foundation.ResultPattern` is permitted
as an optional dependency (when a helper needs to express failure as a value).

### Guard rule — what belongs in commons

A helper belongs in `josyn-commons` if and only if:

- It is reusable across **≥ 2** josyn repos.
- It carries **no JOSYN-domain knowledge** (no sessions, no jobs, no IPC).
- It could theoretically live in **any C# project**, not just JOSYN.

Helpers that do not meet all three criteria stay in the consuming repo.

---

## Consequences

- `josyn-commons` becomes the **sixth repo** in the platform.
- The vocabulary table gains: **Commons** → `josyn-commons` → `JOSYN.Commons.*`
- `josyn-foundation` retains its "stable forever" guarantee — nothing evolving is added there.
- Consuming repos reference `josyn-commons` packages via NuGet, same as foundation packages.
- Package versioning for `josyn-commons` is **independent** of `josyn-foundation`.
- `josyn-platform` documentation and `architecture/naming-conventions.md` are updated to
  reflect the new repo and namespace root.
