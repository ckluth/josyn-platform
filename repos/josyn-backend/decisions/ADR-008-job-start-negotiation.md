# ADR-008 — Job Start Negotiation (Accept / Reject)

**Date:** 2026-06-12
**Status:** Placeholder — not yet specified

---

## Context

ADR-007 establishes that after `JAPServer` spawns `job.exe` and the named pipes are
connected, a first JAP exchange takes place in which the job either **accepts** or
**rejects** the session. This negotiation must complete inside the Turnstile-protected
start phase (see ADR-007, section 4, step 7).

The protocol design — message format, timeout behaviour, error semantics, and how a
rejection is recorded in `SessionStore` — is deferred to this ADR.

---

## Open questions

- What JAP message(s) constitute the accept/reject handshake?
- What does "rejected" mean semantically (wrong arguments, resource unavailable, …)?
- How is a rejection reflected in `SessionStore` (`ExecutionStatus`)?
- What is the timeout for the negotiation, and how is a timeout treated (implicit reject)?
- Does the job.exe side (JOSYN.JobHost) require changes to support the negotiation?

---

## Decisions

*(to be filled in when this ADR is taken up)*

---

## Consequences

*(to be filled in when this ADR is taken up)*
