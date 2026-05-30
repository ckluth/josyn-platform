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
| **JAP** | The per-session JAP protocol server and shared packages | `josyn-jap` |
| **Foundation** | Cross-cutting infrastructure primitives | `josyn-foundation` |
| **Job Host** | The job execution runtime | `josyn-job-host` |

The documentation/architecture repo is named **`josyn-platform`** (not `josyn-platform-docs`)
to leave room for it to grow beyond pure documentation — ADRs, diagrams, tooling, a future
platform CLI, etc.

---

## Consequences

- No existing repos were renamed (at ADR-001 time) — zero blast radius
- "Platform" is now the correct word when referring to all repos together
- `josyn-jap` (formerly `josyn-system`) carries the JAP protocol server; "JAP" is its vocabulary word
- Future cross-cutting packages or tools should live in `josyn-platform` or use `JOSYN.Platform.*` as their namespace root
- Documentation of the whole platform lives in `josyn-platform`

---

## Amendment — 2026-05-30 (supersedes earlier 2026-05-30 amendment)

The `JOSYN.System` → `JOSYN.Jap` rename raised a follow-on question: should `josyn-job-host`
receive a namespace that reflects its architectural decoupling — i.e., one that does *not*
contain "Jap"?

Three options were considered:

| Option | Namespace | Rejected because |
|--------|-----------|-----------------|
| A | `JOSYN.Jap.JobHost` | See below — **superseded** |
| B | `JOSYN.Frontend.JobHost` | Introduces "Frontend" as a 6th vocabulary word with weak semantic entitlement; the existing five-word vocabulary is deliberately tight |
| C | `JOSYN.JobHost` | — **chosen** |

An earlier revision of this ADR chose Option A and rejected Option C on the grounds that
it breaks the universal three-segment `JOSYN.<Layer>.<Component>` pattern. That reasoning
was correct for platform-internal packages — but missed a critical distinction:

> **`JOSYN.JobHost` is a consumer-facing API, not a platform-internal package.**

Its primary audience is *job authors* who have no knowledge of and no interest in the
"Jap" protocol layer. They link the package, add `[JobEntryPoint]`, call `Core.Run(args)`,
and exit. Exposing "Jap" in every `using` statement, every namespace declaration, and
every attribute reference leaks an internal architectural concept into a public-facing
developer experience.

The repo name `josyn-job-host` (not `josyn-jap-job-host`) already makes this distinction
at the repo boundary. The namespace and package ID follow the same logic.

**Decision: `JOSYN.JobHost` is the package ID and namespace root.**

This establishes a documented rule: packages whose primary audience is *external consumers
outside the JOSYN platform* **omit the `<Layer>` segment** and use the two-segment form
`JOSYN.<Component>`. `JOSYN.JobHost` is the canonical and currently only example.
All platform-internal packages continue to follow `JOSYN.<Layer>.<Component>` without exception.

The rule is recorded in `architecture/naming-conventions.md` under *Repository Names*.

---

## Amendment — 2026-05-29

The word **"system"** (previously used for the JAP session server repo) has been retired.
With "platform" established as the umbrella, "system" became ambiguous — it echoes the legacy
`JobSystem.*` world and no longer names anything precisely.

**`josyn-system` has been renamed to `josyn-jap`** and all `JOSYN.System.*` namespaces to
`JOSYN.Jap.*`. The vocabulary entry **System** → **JAP** reflects this.

GitHub remote: `ckluth/josyn-system` → `ckluth/josyn-jap`  
Local folder: `josyn-system` → `josyn-jap`
