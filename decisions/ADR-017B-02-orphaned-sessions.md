# ADR-017B-02 — Resolving Orphaned Sessions

**Date:** 2026-06-12
**Status:** Placeholder — not yet specified

---

## Context

Several failure paths in the session lifecycle can leave a `SessionStore` record in a
terminal-preparing state with no running job process:

- `JAPServer` spawns `job.exe` but the process fails to start (OS error).
- `job.exe` starts but the named pipes never connect (timeout / crash before IPC).
- The accept/reject negotiation times out or the job rejects (ADR-018B-01).
- `JAPServer` itself crashes after persisting the session but before spawning `job.exe`.
- The host machine loses power or is rebooted mid-session.

In all these cases the session record remains with `ExecutionStatus = "preparing"` and no
`Result`, no exit codes, and no error report — an orphan. The current schema has no
status column expressive enough to distinguish a live preparing session from a dead one.

This is a known limitation explicitly noted in the sanity notes for
`JOSYN.Backend.SessionStarter` and referenced in ADR-017B-01.

---

## Open questions

- What `ExecutionStatus` values are needed to distinguish live, failed, completed, and
  orphaned sessions?
- Is a dedicated `ExecutionStatus` column sufficient, or does the schema need additional
  columns (e.g. `JAPServerStartedAt`, `JobSpawnedAt`, `NegotiationCompletedAt`)?
- Who is responsible for detecting orphans — a watchdog, a startup scan, or the next
  orchestrator cycle?
- What is the recovery action for an orphan (re-run, alert, discard)?
- Does the schema migration require a new Flyway script (`V004__...`)?
- How does impersonation (ADR-017B-01) affect the process that performs orphan detection
  (it may run under a different account)?

---

## Decisions

*(to be filled in when this ADR is taken up)*

---

## Consequences

*(to be filled in when this ADR is taken up)*
