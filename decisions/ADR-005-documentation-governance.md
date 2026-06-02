# ADR-005 — Documentation Governance and Agentic Way of Work

**Date:** 2026-06-02
**Status:** Accepted

---

## Context

`josyn-platform` is the single source of truth for platform architecture and coding standards.
As `josyn-backend` evolves — and as other subject repos grow — each will accumulate its own
sub-architecture, component-level decisions, and operational detail. The question this ADR
answers: does that content stay in `josyn-platform`, or does it migrate into each subject repo?

A second related question: how should agent sessions be bootstrapped when working inside a
subject repo rather than from the platform seat?

---

## Decision

### 1. josyn-platform keeps full documentation authority

`josyn-platform` remains the single source of truth for all architecture documentation —
both platform-wide and repo-specific. No documentation authority is distributed to subject
repos. Subject repos do not define architecture, do not host ADRs, and do not override
platform rules.

The concern that motivated this discussion was not about authority — it was about the
platform repo becoming a confusing mess as it grows. That concern is addressed by structure,
not by distributing authority.

### 2. The platform repo scales via transparent folder structure

When a subject repo's documentation grows beyond a single file, it graduates from a flat
file to a dedicated subfolder under `repos/`:

```
repos/
├── josyn-backend/
│   ├── overview.md          ← current content of josyn-backend.md
│   ├── architecture.md      ← backend-internal sub-architecture (when needed)
│   └── decisions/           ← backend-specific ADRs (when needed)
├── josyn-foundation.md      ← single file while small
├── josyn-jap.md
└── ...
```

The `architecture/` tree at the repo root remains **platform-wide only** — it never
contains content scoped to a single subject repo. The `repos/<name>/` subtree is clearly
scoped: documentation about that repo's internals, not the platform at large.

This keeps the platform repo clean and navigable as it grows, without weakening its authority.

### 3. Each subject repo gets its own AGENTS.md

Each subject repo maintains its own `AGENTS.md` for **operational context**: how to build,
what solutions exist, current state, repo-local sanity notes. This file makes an agent
session inside that repo self-sufficient for day-to-day work.

The local `AGENTS.md` is explicitly **subordinate** to `josyn-platform`. It provides
convenience, not authority. It must:

- Direct the agent to `josyn-platform` for all architectural rules, coding standards, and
  sanity check criteria.
- Not duplicate or override any rule defined in `josyn-platform`.
- State its subordinate status clearly at the top.

---

## Consequences

- Agent sessions inside a subject repo are operationally self-sufficient without reading
  all of `josyn-platform` — the local `AGENTS.md` covers what is needed for that repo.
- Architecture authority is never ambiguous: it always lives in `josyn-platform`.
- The `repos/` folder in `josyn-platform` grows organically — no restructuring is needed
  until a subject repo's documentation justifies a subfolder.
- `josyn-platform` remains the natural starting point (the "agentic pilot seat") for any
  cross-cutting or architectural work.
