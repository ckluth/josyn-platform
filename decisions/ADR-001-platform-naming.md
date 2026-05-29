# ADR-001 — Platform Naming Vocabulary

**Date:** 2026-05-29
**Status:** Accepted

---

## Context

The JOSYN ecosystem consists of multiple repos. As the repo count grew, the word "system"
was doing double duty:

1. **`josyn-system`** (a specific repo) — the orchestration backend (JAPServer, shared contracts, logging)
2. **"the JOSYN system"** (colloquial) — the entire platform including all repos

A fourth repo for cross-cutting architecture documentation (`josyn-system-docs`) was proposed.
This surfaced the collision: would `josyn-system-docs` document the `josyn-system` *repo*,
or the *whole platform*?

Additionally, `josyn-job-host` was deliberately kept outside the "system" naming to signal
its architectural decoupling. This made "system" carry a specific, narrow meaning — which
made the wide-scope collision worse.

---

## Decision

Introduce **"platform"** as the umbrella concept word for the entire JOSYN ecosystem.

The vocabulary map:

| Word | Scope | Repo |
|------|-------|------|
| **Platform** | The entire JOSYN ecosystem | `josyn-platform` (this repo) |
| **System** | The orchestration backend only | `josyn-system` |
| **Foundation** | Cross-cutting infrastructure primitives | `josyn-foundation` |
| **Job Host** | The job execution runtime | `josyn-job-host` |

The documentation/architecture repo is named **`josyn-platform`** (not `josyn-platform-docs`)
to leave room for it to grow beyond pure documentation — ADRs, diagrams, tooling, a future
platform CLI, etc.

---

## Consequences

- No existing repos are renamed — zero blast radius
- "Platform" is now the correct word when referring to all four repos together
- `josyn-system` retains its name; "system" continues to mean the orchestration backend
- Future cross-cutting packages or tools should live in `josyn-platform` or use `JOSYN.Platform.*` as their namespace root
- Documentation of the whole platform lives in `josyn-platform`
