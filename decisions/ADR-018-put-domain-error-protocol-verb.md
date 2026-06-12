# ADR-018 — JAP Protocol: Separate Verb for Domain-Error Outcomes

**Date:** 2026-06-12
**Status:** Not Accepted

---

## Context

ADR-016 defines two distinct terminal statuses for job sessions that complete without
an unhandled exception:

| Status | Meaning |
|--------|---------|
| `finished-successfully` | Job ran cleanly and its outcome is a success |
| `finished-with-errors` | Job ran cleanly but its domain outcome is an error |

The distinction was introduced deliberately: `finished-faulted` captures infrastructure or
runtime failures (`PutError` called), while `finished-with-errors` captures deliberate
domain-level failure — the job reached a conclusion, but that conclusion was negative.

The open question is: **how does the JAPServer learn which of these two statuses to set?**

### What `PutRawResult` is

`IJosynApplicationProtocol.PutRawResult(string payload)` transmits an opaque,
unschematized byte string from the job process to the JAPServer. The payload is the
job's result record serialized by `PropertyBag`. It carries no schema contract that
the JAPServer can reliably interpret; its structure is entirely the job author's concern.

Peeking inside the payload to look for a `Succeeded` field was considered and rejected:

- No schema contract → any convention is fragile.
- The payload field `Succeeded` is entirely optional (demo job happens to have one).
- Coupling the transport layer to payload content violates the clean separation between
  the JAP protocol and the job's domain model.

---

## Decision

Add a second protocol verb to `IJosynApplicationProtocol`:

```csharp
/// <summary>
/// Reports a domain-level error outcome. The job ran to completion but its
/// subject-matter conclusion is negative. Maps to ExecutionStatus.FinishedWithErrors.
/// The optional payload carries a human-readable description or a serialized
/// error record for diagnostics — it is NOT a job result record.
/// </summary>
Task<Result> PutDomainError(string? description = null);
```

The two verbs then have unambiguous, exclusive semantics:

| Call | JAPServer sets |
|------|---------------|
| `PutRawResult(payload)` | `FinishedSuccessfully` |
| `PutDomainError(description?)` | `FinishedWithErrors` |
| `PutError(message, ex?)` | `FinishedFaulted` |

A job that has a domain-level failure calls `PutDomainError(...)` instead of
`PutRawResult(...)`. No result payload is expected in the error case; if the job
wants to communicate detail it passes a description string.

---

## Rationale

**Option A — `PutRawResult` implicitly means success, `PutError` for domain failures**
was rejected because `PutError` is semantically tied to infrastructure faults (unhandled
exceptions). Routing deliberate domain failures through it would collapse the
`finished-faulted` / `finished-with-errors` distinction back to a single state.

**Option B — `PutRawResult(string payload, bool succeeded = true)`**
was rejected because it encodes the outcome in a boolean flag on what is otherwise a
data-transport call. The flag is easy to overlook and its meaning is not self-evident
at the call site. It also couples success-signalling to the result-payload call, which
is conceptually separate.

**Option C — separate verb `PutDomainError`** (this decision)
is preferred because:

- The verb name at the call site is unambiguous: `japClient.PutDomainError("No records found")`.
- The result-transport path (`PutRawResult`) and the error-signal path (`PutDomainError`)
  remain independent — a job cannot accidentally combine them.
- The JAP protocol gains a new verb rather than overloading an existing one with a flag.
- Payload semantics are different: `PutRawResult` carries a structured result record;
  `PutDomainError` carries an optional human-readable description.

---

## Consequences

### Affected components

| Component | Change |
|-----------|--------|
| `JOSYN.Jap.Contract` — `IJosynApplicationProtocol` | Add `PutDomainError` method |
| `JOSYN.Jap.JAPServer` — `JAPServer.cs` | Implement `PutDomainError`; set `FinishedWithErrors` |
| `JOSYN.Jap.JAPServer` — `Host.cs` | `PutRawResult` path sets `FinishedSuccessfully` (not `PutDomainError` path) |
| `JOSYN.JobHost` — `JAPClient.cs` | Implement `PutDomainError` client call |
| `josyn-jap` repository | New package version required (contract change) |

### Guard: mutual exclusivity

`PutRawResult` and `PutDomainError` are mutually exclusive within a single session.
The JAPServer should log a warning (and ignore the second call) if both are invoked —
consistent with the existing guard behaviour on `PutError`.

### `FinishedWithErrors` reachability

Before this decision is implemented, `FinishedWithErrors` is **unreachable** at runtime.
That is intentional — the status is defined and reserved, but the transport mechanism
is not yet in place. The implementation of this ADR is the gating item.

---

## Open Questions (to resolve before accepting)

1. **Naming**: `PutDomainError` vs `ReportDomainFailure` vs `PutFailure`?
   Current preference: `PutDomainError` — consistent with the `Put*` verb family already
   in `IJosynApplicationProtocol`.

2. **Payload type**: `string? description` is deliberately unstructured.
   Should it accept a serialized PropertyBag record for richer diagnostics? Decision:
   leave as `string?` for now — structured error payloads can be a future extension.
