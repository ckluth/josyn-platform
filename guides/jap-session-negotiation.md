# JAP Session Negotiation

> **Audience:** Developers who are new to the JOSYN platform and want to understand how the
> JAP Server and a job executable agree on whether a session should actually run.

---

## The Problem

When the scheduler decides to start a job, it cannot know in advance whether the job executable
itself is willing to run right now. A job may want to look at its current environment — for
example, query how many other instances of itself are already running — and only then decide
whether to accept or decline the session. The scheduler must not simply assume the job will
always run.

At the same time, the session record must be persisted in the database *before* the job is
spawned (so it is visible to operators and monitoring), but it must not be marked `Running`
until the job explicitly confirms it wants to proceed.

This creates a two-phase handshake: the JAP Server and the job executable must *negotiate*
before the session is considered in-flight.

---

## The Actors

| Actor | Role |
|-------|------|
| **JAP Server** (`JAPServer` / `Host.Starter`) | Orchestrates the session. Owns the database record and the named-pipe server. |
| **Job executable** | Consumer. Connects to the pipe, calls protocol methods, and decides accept or reject. |
| **`IJosynApplicationProtocol`** | The shared contract: the set of pipe-callable methods both sides agree on. |

---

## The Status Lifecycle

```
[Scheduler decides to start a job]
            |
            v
      [ Preparing ]   <-- Session record created in DB
            |
     +------+------+
     |             |
  Accept     Reject / Timeout
     |             |
     v             v
  [Running]  [FinishedRejected]
     |
     v
  FinishedSuccessfully / FinishedFaulted / FinishedWithErrors
```

---

## Step-by-Step Walk-through

### 1. Session record created (`Preparing`)

Inside `Host.Starter.Prepare()`, a `JobSessionRecord` is written to the database with
`ExecutionStatus.Preparing`. This happens *before* the job executable is spawned. At this
point the session is visible but not yet active.

```csharp
// Host.Starter.cs
sessionStore.SaveNewSession(new JobSessionRecord
{
    ...
    ExecutionStatus = ExecutionStatus.Preparing,
    ...
});
```

### 2. Pipe server starts, job executable is spawned

`PipesServer.RunAsync(...)` opens a named pipe and launches the job executable. The session
GUID is passed as the connection key so the job knows which session it belongs to.

### 3. The negotiation gate

`JAPServer` holds a `TaskCompletionSource<bool>` called `_negotiationGate`. It starts
unresolved — nobody knows yet whether the session will be accepted or rejected.

```csharp
// JAPServer.cs
private readonly TaskCompletionSource<bool> _negotiationGate =
    new(TaskCreationOptions.RunContinuationsAsynchronously);

internal Task<bool> NegotiationOutcome => _negotiationGate.Task;
```

`Host.Starter` awaits this task with a **30-second timeout**:

```csharp
// Host.Starter.cs
var winner = await Task.WhenAny(ctx.JapServer.NegotiationOutcome, Task.Delay(TimeSpan.FromSeconds(30)));
```

### 4. The job executable decides

The job executable connects to the pipe and calls one of two protocol methods:

**Accept** — the job is ready to run:
```csharp
// JAPServer.cs — called via the JAP pipe
Task<Result> IJosynApplicationProtocol.AcceptSession()
{
    SetStatus(ExecutionStatus.Running);     // DB: Preparing → Running
    _negotiationGate.TrySetResult(true);   // unblocks Host.Starter
    return Task.FromResult(Result.Success);
}
```

**Reject** — the job declines (e.g. too many concurrent instances):
```csharp
// JAPServer.cs — called via the JAP pipe
Task<Result> IJosynApplicationProtocol.RejectSession()
{
    SetTerminalStatus(ExecutionStatus.FinishedRejected);  // DB: Preparing → FinishedRejected
    _negotiationGate.TrySetResult(false);                 // unblocks Host.Starter
    return Task.FromResult(Result.Success);
}
```

A job that wants to inspect concurrent sessions before deciding can call
`GetConcurrentSessionArguments()` first — this also goes through the same pipe.

### 5. Host.Starter reacts

Once `_negotiationGate` resolves, `Host.Starter` falls out of `Task.WhenAny` and checks the
outcome:

- **Accepted (`true`):** `ctx.NegotiationAccepted = true`. The Turnstile releases.
  `Host.Starter` awaits the server loop — the session is now in-flight.
- **Rejected (`false`):** `ctx.ShouldCancelServer = true`. The pipe server is wound down.
  The method returns cleanly; the process exits with code 0.
- **Timeout (no result within 30 s):** The session is forcibly set to `FinishedRejected`,
  the server is cancelled, and a warning is logged.

### 6. The Turnstile

`Prepare()` runs inside `Turnstile.RunAsync(jobName, ...)`. The Turnstile ensures that for a
given job type, only one session can pass through the accept/reject negotiation at a time.
This prevents a race where two simultaneous starts both read "0 concurrent sessions" and both
accept — a concurrency check that would otherwise be inherently racy.

The Turnstile is released only after `ctx.NegotiationAccepted` is set — i.e., after the
database status is already `Running`. By that point, any subsequent start for the same job
type will correctly see an active session.

---

## Why `SetStatus` vs `SetTerminalStatus`?

Both write to the database, but they are semantically different:

| Method | Sets `Finished` timestamp | Sets `TerminalStatusSet` flag |
|--------|--------------------------|-------------------------------|
| `SetStatus` | No | No |
| `SetTerminalStatus` | Yes | Yes |

`SetStatus` is used only for the `Preparing → Running` transition because `Running` is not
a terminal state. `SetTerminalStatus` is used for all states that close the session
(`FinishedSuccessfully`, `FinishedFaulted`, `FinishedRejected`, `FinishedWithErrors`).

The `TerminalStatusSet` flag lets `Host.Starter` know whether it still needs to set a
terminal status after the server loop exits, or whether `JAPServer` already did it via a
protocol call (e.g. `PutError` or `PutDomainError`).

---

## Summary

```
Host.Starter                    JAPServer                   Job executable
     |                               |                            |
     |-- SaveNewSession(Preparing) ->|                            |
     |-- PipesServer.RunAsync ------>|--------- spawn ----------->|
     |-- await NegotiationOutcome -->|                            |
     |        (30 s timeout)         |<-- AcceptSession() --------|
     |                               |-- SetStatus(Running)       |
     |                               |-- gate.SetResult(true)     |
     |<------------------------------|                            |
     |  (Turnstile releases)         |                            |
     |-- await ServerTask ---------->|<--- protocol calls --------|
     |                               |                            |
     |  (server loop ends)           |                            |
     |-- SetTerminalStatus --------->|                            |
```

The key insight: **the job executable is the decision-maker**. The JAP Server provides the
gate (`TaskCompletionSource`) and the database plumbing, but it never decides on its own
whether to accept or reject. That responsibility is entirely delegated to the job — and
the 30-second timeout is the safety net for jobs that fail to decide at all.
