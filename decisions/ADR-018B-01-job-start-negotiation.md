# ADR-018B-01 — Job Start Negotiation (Accept / Reject)

**Date:** 2026-06-13
**Status:** Accepted

---

## Context

ADR-017B-01 establishes that after `JAPServer` spawns `job.exe` and the named pipes are
connected, a first JAP exchange takes place in which the job either **accepts** or
**rejects** the session. This negotiation must complete inside the Turnstile-protected
start phase (see ADR-017B-01, section 4, step 6).

The purpose of this negotiation is **parallel execution control**: only the job itself
can define whether and when multiple instances of the same job type may run simultaneously.
The Turnstile serialises concurrent start attempts for the same job type; the negotiation
is the window in which the job evaluates its own policy and declares the outcome.

---

## Decision

### 1. Two new protocol verbs — negotiation outcome

```csharp
/// <summary>
/// Signals that the job accepts the session. Execution proceeds normally.
/// JAPServer transitions ExecutionStatus from <c>preparing</c> to <c>running</c>.
/// </summary>
Task<Result> AcceptSession();

/// <summary>
/// Signals that the job rejects the session. Execution does not proceed.
/// JAPServer transitions ExecutionStatus from <c>preparing</c> to <c>finished-rejected</c>.
/// </summary>
Task<Result> RejectSession();
```

Exactly one of these must be called as the very first JAP exchange, before
`GetRawArguments` and before any other protocol call.

### 2. One new protocol verb — concurrent session arguments

```csharp
/// <summary>
/// Returns the raw arguments of all currently running sessions of the same job type.
/// Used by JobHost to evaluate a conditional parallel-execution policy.
/// Returns an empty collection when no sibling sessions are running.
/// </summary>
Task<Result<IReadOnlyList<string>>> GetConcurrentSessionArguments();
```

JAPServer resolves this from `SessionStore` — all sessions of the same `JobTypeName`
with a transient `ExecutionStatus` (`running` or `running-cancellation-requested`),
excluding the session being started.

### 3. Parallel execution policy — three tiers

The job defines its parallel execution policy. JobHost evaluates it and calls
`AcceptSession` or `RejectSession` accordingly.

| Tier | How defined | Behaviour |
|------|------------|-----------|
| **Default** | Nothing declared | Never parallel — reject if any sibling session is currently running |
| **Always allowed** | `[ParallelExecutionAlwaysAllowed]` attribute on the entry point | Always accept regardless of running siblings |
| **Conditional** | `[BeforeJobEntry]` method sets a predicate on `JobRunner<TArguments>` | JobHost fetches sibling arguments via `GetConcurrentSessionArguments`, evaluates predicate, accepts or rejects |

**"Never parallel" is the safe default.** A job that declares nothing cannot accidentally
run in parallel. Parallel execution is always opt-in.

### 4. JobHost evaluation sequence

```
pipes connected
    │
    ├─ 1. Evaluate parallel-execution policy (attribute or [BeforeJobEntry] hook)
    │       ├─ Always allowed  → AcceptSession()
    │       ├─ Default (never) → GetConcurrentSessionArguments()
    │       │       ├─ empty list → AcceptSession()
    │       │       └─ non-empty → RejectSession()
    │       └─ Conditional     → GetConcurrentSessionArguments()
    │               ├─ predicate returns true  → AcceptSession()
    │               └─ predicate returns false → RejectSession()
    │
    └─ 2. Proceed to GetRawArguments / execute job (accepted path only)
```

### 5. JAPServer timeout

JAPServer waits for `AcceptSession` or `RejectSession` with a **30-second timeout**.
If neither is received within that window — because `job.exe` hung without crashing and
is holding the pipe open — the Turnstile would be held indefinitely, blocking all future
starts of the same job type. The timeout prevents this: it treats the hung session as an
implicit rejection, sets `ExecutionStatus → finished-rejected`, releases the Turnstile,
and exits.

### 6. Turnstile scope

The Turnstile is held from GUID allocation through the accept/reject outcome.
Releasing at spawn time or at pipe connection time would be too early — a rejected job
would vacate the concurrency slot before the next queued start learns of the failure.
The slot is released only when the session is definitively in-flight (`running`) or
definitively closed (`finished-rejected`).

---

## Rationale

1. **Two explicit verbs over one verb with a flag.** `AcceptSession()` and
   `RejectSession()` are self-documenting. A single `ReportStartOutcome(bool)`
   hides the outcome in a flag and invites ambiguity at the call site.

2. **Policy lives in the job, not in the platform.** Only the job knows its own invariants.
   The platform provides the mechanism (the negotiation window and the sibling-arguments
   query); the job provides the policy. This separation means the platform never needs to
   understand job-specific business rules.

3. **"Never parallel" as the safe default.** A job that declares nothing cannot accidentally
   run in parallel. Parallel execution is always a deliberate, explicit opt-in.

4. **`GetConcurrentSessionArguments` over a richer session query.** The job only needs the
   arguments of running siblings to make its decision. Exposing full session records would
   couple the job to the session data model. Raw argument strings are sufficient and keep
   the contract minimal.

5. **Timeout treated as implicit rejection.** A job that never calls either verb has silently
   failed the handshake. `finished-rejected` is the conservative outcome: the session does not
   proceed, the Turnstile releases, and no resources are wasted.

---

## Consequences

- `IJosynApplicationProtocol` gains three new methods: `AcceptSession`, `RejectSession`,
  and `GetConcurrentSessionArguments`. All implementors (`JAPServer`, `JAPClient`) and
  the JIP dispatcher registration must be updated.
- `JobHost` (`Core.Run`) evaluates the parallel execution policy and calls `AcceptSession`
  or `RejectSession` before any other protocol exchange.
- `JAPServer` adds a negotiation wait phase (30-second timeout) between process spawn and
  the JAP serve loop.
- `JAPServer` implements `GetConcurrentSessionArguments` by querying `SessionStore`.
- The `preparing` ExecutionStatus (see ADR-016) covers the entire negotiation window:
  `pending → preparing` on Turnstile entry, `preparing → running` on accept,
  `preparing → finished-rejected` on reject or timeout.
- `josyn-jap` and `josyn-job-host` packages require a version bump (contract change).
